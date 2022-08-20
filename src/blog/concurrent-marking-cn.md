***

标题：“V8 中的并发标记”
作者：“乌兰·德根巴耶夫，迈克尔·利普茨和汉内斯·佩耶尔 - 主线解放者”
化身：

*   '乌兰-德根巴耶夫'
*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2018-06-11 13：33：37
    标签：
*   内部
*   记忆
    描述： '这篇文章描述了一种称为并发标记的垃圾回收技术。
    推文：“1006187194808233985”

***

这篇文章描述了垃圾回收技术，称为*并发标记*.优化允许 JavaScript 应用程序继续执行，而垃圾回收器扫描堆以查找和标记活动对象。我们的基准测试表明，并发标记可将主线程上的标记时间减少 60%–70%。并发标记是最后一块拼图[奥里诺科项目](/blog/orinoco)— 以增量方式将旧的垃圾回收器替换为新的大多数并发和并行垃圾回收器的项目。默认情况下，并发标记在 Chrome 64 和 Node.js v10 中处于启用状态。

## 背景

标记是 V8 的一个阶段[马克-紧凑](https://en.wikipedia.org/wiki/Tracing_garbage_collection)垃圾回收器。在此阶段，收集器发现并标记所有活动对象。标记从一组已知的活动对象开始，例如全局对象和当前活动函数 - 所谓的根。收集器将根标记为活动，并按照其中的指针发现更多活动对象。收集器继续标记新发现的对象并跟踪指针，直到没有更多要标记的对象。在标记结束时，堆上所有未标记的对象都无法从应用程序访问，并且可以安全地回收。

我们可以将标记视为[图形遍历](https://en.wikipedia.org/wiki/Graph_traversal).堆上的对象是关系图的节点。从一个对象到另一个对象的指针是图形的边缘。给定图中的一个节点，我们可以使用[隐藏类](/blog/fast-properties)的对象。

![Figure 1. Object graph](/\_img/concurrent-marking/00.svg)

V8 使用每个对象的两个标记位和一个标记工作列表来实现标记。两个标记位编码三种颜色：白色（`00`），灰色（`10`）和黑色（`11`).最初，所有物体都是白色的，这意味着收藏家还没有发现它们。当收集器发现白色物体并将其推到标记工作清单上时，该物体将变为灰色。当收集器将灰色对象从标记工作清单中弹出并访问其所有字段时，该对象将变为黑色。这种方案称为三色标记。当不再有灰色对象时，标记完成。所有剩余的白色物体都无法访问，可以安全地回收。

![Figure 2. Marking starts from the roots](/\_img/concurrent-marking/01.svg)

![Figure 3. The collector turns a grey object into black by processing its pointers](/\_img/concurrent-marking/02.svg)

![Figure 4. The final state after marking is finished](/\_img/concurrent-marking/03.svg)

请注意，上述标记算法仅在标记过程中暂停应用程序时才有效。如果我们允许应用程序在标记期间运行，则应用程序可以更改图形并最终诱骗收集器释放活动对象。

## 减少打标暂停

对于大型堆，一次执行所有标记可能需要几百毫秒。

![](/\_img/concurrent-marking/04.svg)

这种长时间的暂停可能会使应用程序无响应，并导致用户体验不佳。2011年，V8从停止世界标记切换到增量标记。在增量标记期间，垃圾回收器将标记工作拆分为较小的块，并允许应用程序在块之间运行：

![](/\_img/concurrent-marking/05.svg)

垃圾回收器选择在每个块中执行多少增量标记工作，以匹配应用程序的分配速率。在常见情况下，这大大提高了应用程序的响应能力。对于内存压力下的大型堆，当收集器尝试跟上分配时，仍可能出现长时间的暂停。

增量标记不是免费的。应用程序必须通知垃圾回收器有关更改对象图的所有操作。V8 使用 Dijkstra 样式的写入屏障实现通知。表单的每次写入操作之后`object.field = value`在 JavaScript 中，V8 插入了写入屏障代码：

```cpp
// Called after `object.field = value`.
write_barrier(object, field_offset, value) {
  if (color(object) == black && color(value) == white) {
    set_color(value, grey);
    marking_worklist.push(value);
  }
}
```

写入屏障强制执行不变量，即没有黑色对象指向白色对象。这也称为强三色不变量，并保证应用程序无法从垃圾回收器中隐藏活动对象，因此标记结束时的所有白色对象对于应用程序来说都是真正无法访问的，并且可以安全地释放。

增量标记与空闲时间垃圾回收计划很好地集成在一起，如[早期的博客文章](/blog/free-garbage-collection).Chrome的Blink任务计划程序可以在主线程的空闲时间内安排小的增量标记步骤，而不会导致卡顿。如果空闲时间可用，则此优化效果非常好。

由于写入屏障成本，增量标记可能会降低应用程序的吞吐量。可以通过使用其他工作线程来提高吞吐量和暂停时间。有两种方法可以在工作线程上进行标记：并行标记和并发标记。

**平行**标记发生在主线程和工作线程上。应用程序在整个并行标记阶段暂停。它是停止世界标记的多线程版本。

![](/\_img/concurrent-marking/06.svg)

**并发的**标记主要发生在工作线程上。在进行并发标记时，应用程序可以继续运行。

![](/\_img/concurrent-marking/07.svg)

以下两节介绍我们如何在 V8 中添加对并行和并发标记的支持。

## 平行打标

在并行标记期间，我们可以假设应用程序未同时运行。这大大简化了实现，因为我们可以假设对象图是静态的并且不会更改。为了并行标记对象图，我们需要使垃圾回收器数据结构线程安全，并找到一种在线程之间有效共享标记工作的方法。下图显示了并行标记中涉及的数据结构。箭头指示数据流的方向。为简单起见，该图省略了堆碎片整理所需的数据结构。

![Figure 5. Data structures for parallel marking](/\_img/concurrent-marking/08.svg)

请注意，线程仅从对象图中读取，从不更改它。对象的标记位和标记工作列表必须支持读取和写入访问。

## 标记工作清单和工作窃取

标记工作列表的实现对于性能至关重要，并在快速线程本地性能与在其他线程用完工作时可以分配给其他线程的工作量之间取得平衡。

权衡空间的极端方面是（a）使用完全并发的数据结构以获得最佳共享，因为所有对象都可以共享，以及（b）使用完全线程本地的数据结构，其中没有对象可以共享，优化线程本地吞吐量。图 6 显示了 V8 如何通过使用基于线程本地插入和删除段的标记工作列表来平衡这些需求。一旦段已满，它就会发布到可用于窃取的共享全局池。通过这种方式，V8 允许标记线程尽可能长时间地在本地运行而无需任何同步，并且仍然处理单个线程到达新的对象子图的情况，而另一个线程在完全耗尽其本地段时会挨饿。

![Figure 6. Marking worklist](/\_img/concurrent-marking/09.svg)

## 并发标记

并发标记允许 JavaScript 在主线程上运行，而工作线程正在访问堆上的对象。这为许多潜在的数据竞赛打开了大门。例如，JavaScript 可能在工作线程读取字段的同时写入该字段。数据争用可能会混淆垃圾回收器以释放活动对象或将基元值与指针混淆。

主线程上更改对象图的每个操作都是数据争用的潜在源。由于 V8 是一个高性能引擎，具有许多对象布局优化，因此潜在数据竞跑源的列表相当长。下面是一个高级细分：

*   对象分配。
*   写入对象字段。
*   对象布局更改。
*   从快照反序列化。
*   函数解优化期间的实例化。
*   年轻一代垃圾收集期间的疏散。
*   代码修补。

主线程需要与这些操作上的工作线程同步。同步的成本和复杂性取决于操作。大多数操作允许与原子内存访问进行轻量级同步，但少数操作需要对对象进行独占访问。在下面的小节中，我们将重点介绍一些有趣的案例。

### 写入障碍

通过对对象字段的写入操作转换为[轻松的原子写入](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering)并调整写入障碍：

```cpp
// Called after atomic_relaxed_write(&object.field, value);
write_barrier(object, field_offset, value) {
  if (color(value) == white && atomic_color_transition(value, white, grey)) {
    marking_worklist.push(value);
  }
}
```

将其与以前使用的写入屏障进行比较：

```cpp
// Called after `object.field = value`.
write_barrier(object, field_offset, value) {
  if (color(object) == black && color(value) == white) {
    set_color(value, grey);
    marking_worklist.push(value);
  }
}
```

有两个更改：

1.  源对象的颜色检查 （`color(object) == black`） 消失了。
2.  的颜色过渡`value`从白色到灰色以原子方式发生。

如果没有源对象颜色检查，写入障碍变得更加保守，即即使这些对象不是真正可访问的，它也可能将对象标记为活动。我们删除了检查，以避免在写入操作和写入屏障之间需要的昂贵内存栅栏：

```cpp
atomic_relaxed_write(&object.field, value);
memory_fence();
write_barrier(object, field_offset, value);
```

如果没有内存围栏，可以在写入操作之前对对象颜色加载操作进行重新排序。如果我们不阻止重新排序，则写入屏障可能会观察到灰色对象的颜色并纾困，而工作线程则在看不到新值的情况下标记对象。Dijkstra等人提出的原始书写屏障也不检查物体颜色。他们这样做是为了简单，但我们需要它来保持正确性。

### 救助工作清单

某些操作（例如代码修补）需要对对象进行独占访问。早期，我们决定避免每个对象锁，因为它们可能导致优先级反转问题，其中主线程必须等待在持有对象锁时取消调度的工作线程。我们不是锁定对象，而是允许工作线程从访问对象中解救出来。工作线程通过将对象推送到救助工作列表中来实现此目的，该列表仅由主线程处理：

![Figure 7. The bailout worklist](/\_img/concurrent-marking/10.svg)

工作线程可以拯救优化的代码对象、隐藏类和弱集合，因为访问它们需要锁定或昂贵的同步协议。

回想起来，救助工作清单对增量发展是件好事。我们开始实现工作线程拯救所有对象类型，并逐个添加并发性。

### 对象布局更改

对象的字段可以存储三种类型的值：标记指针、标记的小整数（也称为 Smi）或未标记的值（如未装箱的浮点数）。[指针标记](https://en.wikipedia.org/wiki/Tagged_pointer)是一种众所周知的技术，它允许有效地表示未装箱的整数。在 V8 中，标记值的最低有效位指示它是指针还是整数。这依赖于指针与单词对齐的事实。有关字段是已标记还是未标记的信息存储在对象的隐藏类中。

V8 中的某些操作通过将对象字段转换为另一个隐藏类，将对象字段从已标记更改为未标记（反之亦然）。这样的对象布局更改对于并发标记是不安全的。如果更改发生在工作线程使用旧的隐藏类并发访问对象时，则可能有两种 bug。首先，工作线程可能会错过指针，认为这是一个未标记的值。写入屏障可防止此类错误。其次，工作线程可能会将未标记的值视为指针并取消引用它，这将导致无效的内存访问，通常后跟程序崩溃。为了处理这种情况，我们使用在对象的标记位上进行同步的快照协议。该协议涉及两个方面：将对象字段从标记更改为未标记的主线程，以及访问对象的工作线程。在更改字段之前，主线程确保对象标记为黑色，并将其推送到救助工作列表中，以便以后访问：

```cpp
atomic_color_transition(object, white, grey);
if (atomic_color_transition(object, grey, black)) {
  // The object will be revisited on the main thread during draining
  // of the bailout worklist.
  bailout_worklist.push(object);
}
unsafe_object_layout_change(object);
```

如下面的代码片段所示，工作线程首先加载对象的隐藏类，并使用[原子松弛负载操作](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering).然后，它尝试使用原子比较和交换操作将对象标记为黑色。如果标记成功，则意味着快照必须与隐藏类一致，因为主线程在更改对象布局之前将对象标记为黑色。

```cpp
snapshot = [];
hidden_class = atomic_relaxed_load(&object.hidden_class);
for (field_offset in pointer_field_offsets(hidden_class)) {
  pointer = atomic_relaxed_load(object + field_offset);
  snapshot.add(field_offset, pointer);
}
if (atomic_color_transition(object, grey, black)) {
  visit_pointers(snapshot);
}
```

请注意，经历不安全布局更改的白色对象必须在主线程上标记。不安全的布局更改相对罕见，因此这对实际应用程序的性能没有太大影响。

## 将它们放在一起

我们将并发标记集成到现有的增量标记基础结构中。主线程通过扫描根部并填写标记工作列表来启动标记。之后，它会在工作线程上发布并发标记任务。工作线程通过协作排出标记工作列表，帮助主线程更快地进行标记。偶尔，主线程通过处理救助工作清单和标记工作清单来参与标记。一旦标记工作列表变为空，主线程就会完成垃圾回收。在定版过程中，主线程会重新扫描根部，并可能发现更多的白色物体。这些对象在工作线程的帮助下并行标记。

![](/\_img/concurrent-marking/11.svg)

## 结果

我们[现实世界的基准框架](/blog/real-world-performance)显示，在移动设备和桌面设备上，每个垃圾回收周期的主线程标记时间分别减少了约 65% 和 70%。

![Time spent in marking on the main thread (lower is better)](/\_img/concurrent-marking/12.svg)

并发标记还可以减少 Node.js中的垃圾回收卡顿。这一点尤其重要，因为Node.js从未实现空闲时间垃圾回收调度，因此永远无法在非卡顿关键阶段隐藏标记时间。节点.js v10 中提供的并发标记。
