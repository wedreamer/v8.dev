***

标题： “C++的高性能垃圾回收”
作者： '安东·比基涅夫， 奥马尔·卡茨 （[@omerktz](https://twitter.com/omerktz)），以及迈克尔·利普茨（[@mlippautz](https://twitter.com/mlippautz)），C++记忆低语者
化身：

*   “安东-比基涅夫”
*   'omer-katz'
*   “迈克尔-利普茨”
    日期： 2020-05-26
    标签：
*   内部
*   记忆
*   断续器
    描述： '这篇文章描述了Oilpan C++垃圾回收器，它在Blink中的用法，以及它如何优化扫描，即回收无法访问的内存。
    推文：“1265304883638480899”

***

在过去，我们有[已经](https://v8.dev/blog/trash-talk) [一直](https://v8.dev/blog/concurrent-marking) [写作](https://v8.dev/blog/tracing-js-dom)关于JavaScript的垃圾回收，文档对象模型（DOM），以及如何在V8中实现和优化所有这些。不过，Chromium中并不是所有的东西都是JavaScript，因为大多数浏览器及其嵌入V8的Blink渲染引擎都是用C++编写的。JavaScript 可用于与 DOM 交互，然后由呈现管道处理。

由于围绕DOM的C++对象图与Javascript对象严重纠缠在一起，因此Chromium团队几年前改用垃圾收集器，称为[油锅](https://www.youtube.com/watch?v=\_uxmEyd6uxo)，用于管理此类内存。Oilpan是用C++编写的垃圾回收器，用于管理可以连接到V8的C++内存，使用[跨组件跟踪](https://research.google/pubs/pub47359/)它将纠缠不清的C++/JavaScript 对象图视为一个堆。

这篇文章是一系列 Oilpan 博客文章中的第一篇，这些文章将概述 Oilpan 的核心原则及其C++ API。在这篇文章中，我们将介绍一些支持的功能，解释它们如何与垃圾回收器的各个子系统进行交互，并深入研究并发回收清扫器中的对象。

最令人兴奋的是，Oilpan目前在Blink中实现，但以V8的形式转移到V8。[垃圾回收库](https://chromium.googlesource.com/v8/v8.git/+/HEAD/include/cppgc/).目标是使所有 V8 嵌入器以及更多C++开发人员能够轻松使用C++垃圾回收。

## 背景

油盘实现了[标记扫描](https://en.wikipedia.org/wiki/Tracing_garbage_collection)垃圾回收器，其中垃圾回收分为两个阶段：*标记*其中，将扫描托管堆以查找活动对象，以及*席卷*其中，将回收托管堆上的死对象。

在介绍时，我们已经介绍了标记的基础知识[V8 中的并发标记](https://v8.dev/blog/concurrent-marking).回顾一下，扫描所有对象以查找活动对象可以看作是图形遍历，其中对象是节点，对象之间的指针是边缘。遍历从根目录开始，这些根是寄存器、本机执行堆栈（从现在开始我们将称之为堆栈）和其他全局变量，如前所述[这里](https://v8.dev/blog/concurrent-marking#background).

在这方面，C++与JavaScript没有什么不同。然而，与JavaScript相反，C++对象是静态类型的，因此无法在运行时更改其表示形式。使用 Oilpan 管理C++对象利用这一事实，并通过访问者模式提供指向其他对象（图中的边缘）的指针的描述。描述 Oilpan 对象的基本模式如下：

```cpp
class LinkedNode final : public GarbageCollected<LinkedNode> {
 public:
  LinkedNode(LinkedNode* next, int value) : next_(next), value_(value) {}
  void Trace(Visitor* visitor) const {
    visitor->Trace(next_);
  }
 private:
  Member<LinkedNode> next_;
  int value_;
};

LinkedNode* CreateNodes() {
  LinkedNode* first_node = MakeGarbageCollected<LinkedNode>(nullptr, 1);
  LinkedNode* second_node = MakeGarbageCollected<LinkedNode>(first_node, 2);
  return second_node;
}
```

在上面的例子中，`LinkedNode`由 Oilpan 管理，如继承自`GarbageCollected<LinkedNode>`.当垃圾回收器处理对象时，它会通过调用`Trace`对象的方法。类型`Member`是一个智能指针，在语法上类似于例如`std::shared_ptr`，它由 Oilpan 提供，用于在标记期间遍历图形时保持一致的状态。所有这些都使Oilpan能够精确地知道指针在其托管对象中的位置。

狂热的读者可能注意到了~~，也可能害怕~~`first_node`和`second_node`在上面的示例中，它们作为原始C++指针保存在堆栈上。Oilpan 不会添加用于处理堆栈的抽象，而是在处理根目录时仅依靠保守的堆栈扫描来查找其托管堆中的指针。这可以通过逐个字迭代堆栈并将这些单词解释为指向托管堆的指针来实现。这意味着 Oilpan 不会对访问堆栈分配的对象施加性能损失。相反，它会将成本转移到垃圾回收时间，在垃圾回收时间，从而保守地扫描堆栈。集成在渲染器中的 Oilpan 会尝试延迟垃圾回收，直到它达到保证没有有趣堆栈的状态。由于Web是基于事件的，并且执行是由事件循环中的处理任务驱动的，因此这样的机会很多。

Oilpan用于Blink，这是一个大型C++代码库，具有许多成熟的代码，因此还支持：

*   通过 mixins 进行多重继承，并引用此类 mixins（内部指针）。
*   在执行构造函数期间触发垃圾回收。
*   通过以下方式使对象从非托管内存中保持活动状态`Persistent`被视为根的智能指针。
*   集合涵盖顺序（例如矢量）和关联（例如集合和映射）容器的集合，并压缩集合支持。
*   弱引用、弱回调和[瞬时](https://en.wikipedia.org/wiki/Ephemeron).
*   在回收单个对象之前执行的终结器回调。

## 扫地C++

请继续关注有关Oilpan中标记如何详细工作的单独博客文章。在本文中，我们假设标记已完成，并且Oilpan在其帮助下发现了所有可到达的对象`Trace`方法。标记所有可访问对象后，设置其标记位。

扫描现在是回收死对象（在标记期间无法访问的对象）的阶段，并且其底层内存要么返回到操作系统，要么可用于后续分配。在下文中，我们将展示 Oilpan 的清扫车的工作原理，既从用途和约束的角度来看，也从它如何实现高回收吞吐量。

清扫器通过迭代堆内存并检查标记位来查找死对象。为了保留C++语义，扫地器必须在释放每个死对象的内存之前调用其析构函数。非平凡的析构函数作为终结器实现。

从程序员的角度来看，没有定义析构函数的执行顺序，因为扫地机使用的迭代不考虑构造顺序。这施加了一个限制，即不允许终结器接触其他堆上对象。对于编写需要完成顺序的用户代码来说，这是一个常见的挑战，因为托管语言通常不支持其最终化语义（例如Java）中的顺序。Oilpan使用Clang插件，该插件静态验证在对象销毁期间是否未访问堆对象：

```cpp
class GCed : public GarbageCollected<GCed> {
 public:
  void DoSomething();
  void Trace(Visitor* visitor) {
    visitor->Trace(other_);
  }
  ~GCed() {
    other_->DoSomething();  // error: Finalizer '~GCed' accesses
                            // potentially finalized field 'other_'.
  }
 private:
  Member<GCed> other_;
};
```

对于好奇的人：Oilpan为复杂的用例提供了预终结回调，这些用例需要在销毁对象之前访问堆。但是，这样的回调在每个垃圾回收周期上比析构函数施加更多的开销，并且仅在Blink中谨慎使用。

## 增量和并发扫描

现在我们已经介绍了托管C++环境中析构函数的限制，现在是时候更详细地了解Oilpan如何实现和优化扫描阶段了。

在深入研究细节之前，重要的是要回顾一下程序在网络上的一般执行方式。任何执行，例如，JavaScript程序以及垃圾回收，都是通过调度任务从主线程驱动的。[事件循环](https://en.wikipedia.org/wiki/Event_loop).与其他应用程序环境非常相似，渲染器支持并发到主线程的后台任务，以帮助处理任何主线程工作。

从简单开始，Oilpan最初实现了停止世界扫描，它作为垃圾收集终结暂停的一部分运行，中断了主线程上应用程序的执行：

![Stop-the-world sweeping](/\_img/high-performance-cpp-gc/stop-the-world-sweeping.svg)

对于具有软实时约束的应用程序，处理垃圾回收时的决定因素是延迟。停止世界扫描可能会导致长时间的暂停，从而导致用户可见的应用程序延迟。作为减少延迟的下一步，扫描是增量的：

![Incremental sweeping](/\_img/high-performance-cpp-gc/incremental-sweeping.svg)

使用增量方法，扫描被拆分并委派给其他主线程任务。在最好的情况下，这些任务完全在[空闲时间](https://research.google/pubs/pub45361/)，避免干扰任何常规应用程序执行。在内部，清扫器根据页面的概念将工作划分为更小的单元。页面可以处于两种有趣的状态：*待扫除*清扫器仍需要处理的页面，以及*已经横扫*扫地机已处理的页面。分配仅考虑已清扫的页，并将从维护可用内存块列表的空闲列表中重新填充本地分配缓冲区 （LAG）。为了从空闲列表中获取内存，应用程序将首先尝试在已扫描的页面中查找内存，然后尝试通过将扫描算法内联到分配中来帮助处理待扫描的页面，并且仅在没有内存的情况下从操作系统请求新内存。

Oilpan 多年来一直使用增量扫描，但随着应用程序及其生成的对象图变得越来越大，扫描开始影响应用程序性能。为了改进增量扫描，我们开始利用后台任务并发回收内存。有两个基本不变量用于排除执行扫地器的后台任务与分配新对象的应用程序之间的任何数据争用：

*   清扫器仅处理根据定义应用程序无法访问的死存储器。
*   应用程序仅在已扫描的页面上分配，根据定义，这些页面不再由扫描器处理。

这两个不变量都确保对象及其内存不应有竞争者。不幸的是，C++严重依赖于作为终结器实现的析构函数。Oilpan强制终结器在主线程上运行，以帮助开发人员并排除应用程序代码本身中的数据竞争。为了解决这个问题，Oilpan 将对象定格推迟到主线程。更具体地说，每当并发扫波器遇到具有终结器（析构函数）的对象时，它会将其推送到将在单独的终结阶段中处理的终结队列中，该阶段始终在运行应用程序的主线程上执行。具有并发扫描的整体工作流如下所示：

![Concurrent sweeping using background tasks](/\_img/high-performance-cpp-gc/concurrent-sweeping.svg)

由于终结器可能需要访问对象的所有有效负载，因此将相应的内存添加到可用列表会延迟到执行终结器之后。如果未执行终结器，则在后台线程上运行的清除程序会立即将回收的内存添加到可用列表中。

# 结果

背景扫描已在Chrome M78中发布。我们[现实世界的基准框架](https://v8.dev/blog/real-world-performance)显示主线程清扫时间缩短了 25%-50%（平均为 42%）。请参阅下面的一组选定的订单项。

![Main thread sweeping time in milliseconds](/\_img/high-performance-cpp-gc/results.svg)

在主线程上花费的剩余时间用于执行终结器。目前正在进行一项工作，以减少 Blink 中大量实例化对象类型的终结器。令人兴奋的部分是，所有这些优化都是在应用程序代码中完成的，因为在没有终结器的情况下，扫描将自动调整。

请继续关注有关C++垃圾回收以及Oilpan库更新的更多帖子，特别是随着我们向V8的所有用户都可以使用的版本迈进。
