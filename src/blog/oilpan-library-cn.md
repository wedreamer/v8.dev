***

标题： “油盘库”
作者： '安东·比基涅夫， 奥马尔·卡茨 （[@omerktz](https://twitter.com/omerktz)），以及迈克尔·利普茨（[@mlippautz](https://twitter.com/mlippautz)），高效的文件移动器
化身：

*   安东-比基涅夫
*   omer-katz
*   迈克尔-利普茨
    日期： 2021-11-10
    标签：
*   内部
*   记忆
*   断续器
    描述：“V8 附带了 Oilpan，这是一个垃圾回收库，用于托管托管C++内存。
    推文：“1458406645181165574”

***

虽然这篇文章的标题可能建议深入研究一系列围绕油底壳的书籍 - 考虑到油底锅的构造规范，这是一个具有惊人数量文献的主题 - 但我们更仔细地研究了Oilpan，这是一个C++垃圾收集器，自V8 v9.4以来一直作为图书馆托管通过V8托管。

油锅是一个[基于跟踪的垃圾回收器](https://en.wikipedia.org/wiki/Tracing_garbage_collection)，这意味着它通过在标记阶段遍历对象图来确定活动对象。然后，在扫描阶段回收死物，我们有[过去博客](https://v8.dev/blog/high-performance-cpp-gc).这两个阶段都可以交错运行，也可以并行于实际C++应用程序代码。堆对象的引用处理是精确的，对于本机堆栈是保守的。这意味着 Oilpan 知道引用在堆上的位置，但必须扫描内存，假设随机位序列表示堆栈的指针。Oilpan 还支持在没有本机堆栈的情况下运行垃圾回收时对某些对象进行压缩（对堆进行碎片整理）。

那么，通过 V8 将其作为库提供有什么关系呢？

Blink，从WebKit分叉，最初使用引用计数，一个[众所周知的C++代码范例](https://en.cppreference.com/w/cpp/memory/shared_ptr)，用于管理其堆内存。引用计数应该解决内存管理问题，但已知由于周期而容易发生内存泄漏。除了这个固有的问题之外，Blink还遭受了[释放后使用问题](https://en.wikipedia.org/wiki/Dangling_pointer)因为有时由于性能原因会省略引用计数。Oilpan最初是专门为Blink开发的，用于简化编程模型并消除内存泄漏和释放后使用的问题。我们相信Oilpan成功地简化了模型，并使代码更加安全。

在Blink中引入Oilpan的另一个可能不太明显的原因是为了帮助集成到其他垃圾收集系统中，例如V8，它最终在实现[统一的 JavaScript 和 C++ 堆](https://v8.dev/blog/tracing-js-dom)其中，Oilpan负责处理C++对象。随着越来越多的对象层次结构被管理以及与 V8 的更好集成，Oilpan 随着时间的推移变得越来越复杂，团队意识到他们正在重新发明与 V8 垃圾回收器相同的概念并解决相同的问题。Blink 中的集成需要构建大约 30k 个目标，才能实际运行 hello world 垃圾回收测试，以实现统一堆。

2020年初，我们开始了从Blink中雕刻出Oilpan并将其封装到库中的旅程。我们决定在 V8 中托管代码，在可能的情况下重用抽象，并在垃圾回收接口上进行一些春季清理。除了解决上述所有问题外，[图书馆](https://docs.google.com/document/d/1ylZ25WF82emOwmi_Pg-uU6BI1A-mIbX_MG9V87OFRD8/)还将使其他项目能够利用垃圾收集C++。我们在 V8 v9.4 中启动了该库，并从 Chromium M94 开始在 Blink 中启用它。

## 盒子里有什么？

与V8的其余部分类似，Oilpan现在提供了一个[稳定的原料药](https://chromium.googlesource.com/v8/v8.git/+/HEAD/include/cppgc/)和嵌入器可能依赖于常规[V8 约定](https://v8.dev/docs/api).例如，这意味着 API 已正确记录（请参阅[垃圾回收](https://chromium.googlesource.com/v8/v8.git/+/main/include/cppgc/garbage-collected.h#17)），并将经历一个弃用期，以防它们被删除或更改。

Oilpan的核心可作为独立的C++垃圾收集器在`cppgc`命名空间。该设置还允许重用现有的 V8 平台，为托管C++对象创建堆。垃圾回收可以配置为自动运行，集成到任务基础架构中，或者也可以在考虑本机堆栈的情况下显式触发。这个想法是允许只想要托管C++对象的嵌入器避免将V8作为一个整体来处理，请参阅以下内容[你好世界计划](https://chromium.googlesource.com/v8/v8.git/+/main/samples/cppgc/hello-world.cc)例如。此配置的嵌入器是PDFium，它使用Oilpan的独立版本[确保 XFA 的安全](https://groups.google.com/a/chromium.org/g/chromium-dev/c/RAqBXZWsADo/m/9NH0uGqCAAAJ?utm_medium=email\&utm_source=footer)这允许更动态的PDF内容。

方便的是，对Oilpan核心的测试使用此设置，这意味着构建和运行特定的垃圾回收测试只需几秒钟。截至今天，[>400个这样的单元测试](https://source.chromium.org/chromium/chromium/src/+/main:v8/test/unittests/heap/cppgc/)因为油盘的核心存在。该设置还可以作为实验和尝试新事物的游乐场，并可用于验证有关原始性能的假设。

Oilpan库还负责在通过V8使用统一堆运行时处理C++对象，这允许C++和JavaScript对象图的完整纠缠。此配置在 Blink 中用于管理 DOM 的C++内存等。Oilpan还公开了一个特征系统，该系统允许使用对确定活动性具有非常特定需求的类型来扩展垃圾收集器的核心。通过这种方式，Blink可以提供自己的集合库，甚至允许构建JavaScript风格的临时映射（[`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)），C++。我们不建议所有人这样做，但它显示了该系统在需要自定义时的功能。

## 我们将走向何方？

Oilpan 库为我们提供了一个坚实的基础，我们现在可以利用它来提高性能。以前，我们需要在 V8 的公共 API 上指定垃圾回收特定功能才能与 Oilpan 交互，现在我们可以直接实现所需的功能。这允许快速迭代，还可以采取捷径并在可能的情况下提高性能。

我们还看到了直接通过Oilpan提供某些基本容器的潜力，以避免重新发明轮子。这将允许其他嵌入器从以前专门为Blink创建的数据结构中受益。

看到油盘的光明未来，我们想提一下，现有的[`EmbedderHeapTracer`](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-embedder-heap.h;l=75)API 不会得到进一步改进，并且可能会在某个时候被弃用。假设使用此类API的嵌入器已经实现了自己的跟踪系统，那么迁移到Oilpan应该就像在新创建的对象上分配C++对象一样简单。[油盘堆](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-cppgc.h;l=91)然后将其连接到 V8 隔离器。用于建模引用的现有基础结构，例如[`TracedReference`](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-traced-handle.h;l=334)（用于参考 V8）和[内部字段](https://source.chromium.org/chromium/chromium/src/+/main:v8/include/v8-object.h;l=502)（对于从 V8 传出的引用）受 Oilpan 支持。

请继续关注未来更多垃圾回收改进！

遇到问题，或有建议？让我们知道：

*   <oilpan-dev@chromium.org>
*   单轨铁路：[眨眼>GarbageCollection](https://bugs.chromium.org/p/chromium/issues/entry?template=Defect+report+from+user\&components=Blink%3EGarbageCollection)（铬），[油锅](https://bugs.chromium.org/p/v8/issues/entry?template=Defect+report+from+user\&components=Oilpan)（V8）

\[^1]：在[研究文章](https://research.google/pubs/pub48052/).
