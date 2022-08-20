***

标题： '背景汇编'
作者： '[罗斯·麦克罗伊](https://twitter.com/rossmcilroy)，主线程后卫'
化身：

*   “罗斯-麦克罗伊”
    日期： 2018-03-26 13：33：37
    标签：
*   内部
    描述： “从Chrome 66开始，V8在后台线程上编译JavaScript源代码，在典型网站上将主线程编译所花费的时间减少了5%到20%。
    推文：“978319362837958657”

***

TL;DR：从Chrome 66开始，V8在后台线程上编译JavaScript源代码，在典型网站上将主线程上编译所花费的时间减少了5%到20%。

## 背景

自版本 41 起，Chrome 支持[解析后台线程上的 JavaScript 源文件](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html)通过 V8 的[`StreamedSource`](https://cs.chromium.org/chromium/src/v8/include/v8.h?q=StreamedSource\&sq=package:chromium\&l=1389)应用程序接口。这使得 V8 能够在 Chrome 从网络下载文件的第一个块后立即开始解析 JavaScript 源代码，并在 Chrome 通过网络流式传输文件时继续并行解析。这可以提供相当大的加载时间改进，因为 V8 几乎可以在文件完成下载时完成对 JavaScript 的分析。

但是，由于 V8 原始基准编译器的限制，V8 仍然需要返回到主线程以完成解析，并将脚本编译为执行脚本代码的 JIT 机器代码。随着切换到我们的新[点火+涡轮风扇管路](/blog/launching-ignition-and-turbofan)，我们现在也能够将字节码编译移动到后台线程，从而释放Chrome的主线程以提供更流畅，响应更快的Web浏览体验。

## 构建后台线程字节码编译器

V8 的 Ignition 字节码编译器采用[抽象语法树 （AST）](https://en.wikipedia.org/wiki/Abstract_syntax_tree)由解析器作为输入生成，并生成字节码流（`BytecodeArray`） 以及相关的元数据，使 Ignition 解释器能够执行 JavaScript 源代码。

![](/\_img/background-compilation/bytecode.svg)

Ignition的字节码编译器在构建时考虑了多线程，但是在整个编译管道中需要进行许多更改才能启用后台编译。其中一个主要变化是防止编译管道在后台线程上运行时访问 V8 的 JavaScript 堆中的对象。V8 堆中的对象不是线程安全的，因为 Javascript 是单线程的，在后台编译期间可能会被主线程或 V8 的垃圾回收器修改。

编译管道有两个主要阶段访问 V8 堆上的对象：AST 内部化和字节码终结。AST 内部化是一个过程，通过该过程，在 AST 中标识的文本对象（字符串、数字、对象-文本样板等）在 V8 堆上分配，以便在执行脚本时可以由生成的字节码直接使用它们。此过程传统上在解析器构建 AST 后立即发生。因此，编译管道后面有许多步骤依赖于已分配的文本对象。为了启用后台编译，我们在编译字节码之后，在编译管道中稍后移动了AST内部化。这需要对管道的后期阶段进行修改，以访问*生*嵌入在 AST 中的文本值，而不是内部化的堆上值。

字节码定型涉及构建字节码`BytecodeArray`对象，用于执行函数，以及关联的元数据 — 例如，一个`ConstantPoolArray`它存储字节码引用的常量，以及`SourcePositionTable`它将 JavaScript 源代码行和列号映射到字节码偏移量。由于 JavaScript 是一种动态语言，因此这些对象都需要位于 JavaScript 堆中，以便在收集与字节码关联的 JavaScript 函数时能够对其进行垃圾回收。以前，其中一些元数据对象将在字节码编译期间分配和修改，这涉及访问 JavaScript 堆。为了启用后台编译，对Ignition的字节码生成器进行了重构，以跟踪此元数据的详细信息，并推迟在JavaScript堆上分配它们，直到编译的最后阶段。

通过这些更改，几乎所有脚本的编译都可以移动到后台线程，只有简短的AST内部化和字节码完成步骤发生在脚本执行之前的主线程上。

![](/\_img/background-compilation/threads.svg)

目前，只有顶级脚本代码和立即调用的函数表达式 （IIFE） 在后台线程上编译 — 内部函数仍然在主线程上延迟编译（首次执行时）。我们希望将来将后台编译扩展到更多情况。但是，即使有这些限制，后台编译也会使主线程自由化更长时间，使其能够执行其他工作，例如对用户交互做出反应，渲染动画或以其他方式产生更流畅，响应更快的体验。

## 结果

我们使用[现实世界的基准框架](/blog/real-world-performance)跨一组热门网页。

![](/\_img/background-compilation/desktop.svg)

![](/\_img/background-compilation/mobile.svg)

在后台线程上可能发生的编译比例取决于在顶级流式处理脚本编译期间编译的字节码的比例，以及在调用内部函数时延迟编译（这仍然必须发生在主线程上）。因此，在主线程上节省的时间比例各不相同，大多数页面的主线程编译时间减少了5%到20%。

## 四. 今后的步骤

有什么比在后台线程上编译脚本更好的呢？根本不需要编译脚本！除了背景编译，我们还一直在努力改进V8的[代码缓存系统](/blog/code-caching)以扩展 V8 缓存的代码量，从而加快您经常访问的网站的页面加载速度。我们希望尽快为您带来这方面的最新信息。敬请期待！
