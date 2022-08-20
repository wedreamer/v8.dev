***

## 标题： 'WebAssembly 编译管道'&#xA;描述： '本文解释了 V8 的 WebAssembly 编译器以及它们何时编译 WebAssembly 代码。

WebAssembly是一种二进制格式，允许您在Web上高效，安全地从JavaScript以外的编程语言运行代码。在本文档中，我们将深入探讨 V8 中的 WebAssembly 编译管道，并解释如何使用不同的编译器来提供良好的性能。

## 起飞

V8 首先编译一个 WebAssembly 模块及其基准编译器，[起飞](/blog/liftoff).升空是一个[一次性编译器](https://en.wikipedia.org/wiki/One-pass_compiler)，这意味着它迭代WebAssembly代码一次，并立即为每个WebAssembly指令发出机器代码。一次性编译器擅长快速生成代码，但只能应用一小部分优化。事实上，Liftoff可以非常快速地编译WebAssembly代码，每秒10兆字节。一旦 Liftoff 编译完成，编译好的 WebAssembly 模块就会返回到 JavaScript。

## 涡轮风扇

Liftoff在很短的时间内发出相当快的机器代码。但是，由于它独立地为每个WebAssembly指令发出代码，因此优化的空间非常小，例如改进寄存器分配或常见的编译器优化，例如冗余负载消除，强度降低或函数内联。

这就是为什么一旦Liftoff编译完成，V8就会立即开始通过重新编译所有函数来“分层”模块。[涡轮风扇](/docs/turbofan)，V8 中针对 WebAssembly 和 JavaScript 的优化编译器。涡轮风扇是一个[多通道编译器](https://en.wikipedia.org/wiki/Multi-pass_compiler)，这意味着它在发出机器代码之前会构建已编译代码的多个内部表示形式。这些额外的内部表示允许优化和更好的寄存器分配，从而显着加快代码速度。

TurboFan 逐个编译 WebAssembly 模块函数。一旦一个函数完成，它就会立即替换 Liftoff 编译的函数。然后，对该函数的任何新调用都将使用TurboFan生成的新的优化代码，而不是Liftoff代码。但请注意，我们不进行堆栈上替换。这意味着，如果 TurboFan 完成优化了在只有 Liftoff 版本可用时已经调用的函数，它将使用 Liftoff 版本完成其执行。

## 代码缓存

如果 WebAssembly 模块是使用`WebAssembly.compileStreaming`，则 TurboFan 生成的机器代码也会被缓存。当从同一 URL 再次获取同一 WebAssembly 模块时，该模块不会编译，而是从缓存加载。有关代码缓存的详细信息可用[在单独的博客文章中](/blog/wasm-code-caching).

## 调试

如前所述，TurboFan应用了优化，其中许多优化涉及重新排序代码，消除变量甚至跳过整个代码段。这意味着，如果要在特定指令处设置断点，则可能不清楚程序执行实际应从何处停止。换句话说，TurboFan代码不太适合调试。因此，当通过打开 DevTools 开始调试时，所有 TurboFan 代码都会再次替换为 Liftoff 代码（“分层”），因为每个 WebAssembly 指令都恰好映射到机器代码的一个部分，并且所有局部和全局变量都完好无损。

## 分析

为了使事情变得更加混乱，在DevTools中，当“性能”选项卡打开并单击“记录”按钮时，所有代码将再次分层（使用TurboFan重新编译）。“记录”按钮启动性能分析。分析Liftoff代码不会具有代表性，因为它仅在TurboFan尚未完成时使用，并且可能比TurboFan的输出慢得多，TurboFan的输出将在绝大多数时间运行。

## 用于实验的标志

对于实验，V8和Chrome可以配置为仅使用Liftoff或仅使用TurboFan编译WebAssembly代码。甚至可以尝试延迟编译，其中函数仅在首次调用时才进行编译。以下标志启用这些实验模式：

*   仅升空：
    *   在 V8 中，将`--liftoff --no-wasm-tier-up`标志。
    *   在 Chrome 中，禁用 WebAssembly 分层 （`chrome://flags/#enable-webassembly-tiering`） 并启用 WebAssembly 基线编译器 （`chrome://flags/#enable-webassembly-baseline`).

*   仅涡轮风扇：
    *   在 V8 中，将`--no-liftoff --no-wasm-tier-up`标志。
    *   在 Chrome 中，禁用 WebAssembly 分层 （`chrome://flags/#enable-webassembly-tiering`） 并禁用 WebAssembly 基线编译器 （`chrome://flags/#enable-webassembly-baseline`).

*   惰性编译：
    *   延迟编译是一种编译模式，其中函数仅在首次调用时才进行编译。与生产配置类似，该函数首先使用Liftoff（阻止执行）进行编译。Liftoff编译完成后，该函数将在后台使用TurboFan重新编译。
    *   在 V8 中，将`--wasm-lazy-compilation`旗。
    *   在 Chrome 中，启用 WebAssembly lazy compilation （`chrome://flags/#enable-webassembly-lazy-compilation`).

## 编译时

有不同的方法来测量Liftoff和TurboFan的编译时间。在 V8 的生产配置中，Liftoff 的编译时间可以通过测量它所花费的时间从 JavaScript 来测量。`new WebAssembly.Module()`完成，或花费的时间`WebAssembly.compile()`以解决承诺。为了测量TurboFan的编译时间，可以在仅使用TurboFan的配置中执行相同的操作。

![The trace for WebAssembly compilation in Google Earth.](/\_img/wasm-compilation-pipeline/trace.svg)

编译也可以更详细地测量`chrome://tracing/`通过启用`v8.wasm`类别。然后，Liftoff 编译是从开始编译到`wasm.BaselineFinished`事件，涡轮风扇编译结束于`wasm.TopTierFinished`事件。编译本身从`wasm.StartStreamingCompilation`事件`WebAssembly.compileStreaming()`，在`wasm.SyncCompile`事件`new WebAssembly.Module()`，并在`wasm.AsyncCompile`事件`WebAssembly.compile()`分别。Liftoff 编译以`wasm.BaselineCompilation`事件， 涡轮风扇编译与`wasm.TopTierCompilation`事件。上图显示了为 Google 地球记录的跟踪，并突出显示了关键事件。

更详细的跟踪数据可通过`v8.wasm.detailed`类别，除其他信息外，它还提供单个函数的编译时间。
