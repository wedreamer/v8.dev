***

标题： “在 V8 中对 WebAssembly 的实验性支持”
作者：《Seth Thompson， WebAssembly Wrangler》
发布日期：2016-03-15 13：33：37
标签：

*   WebAssembly
    描述： “从今天开始，对WebAssembly的实验性支持在V8和Chromium中可用。

***

*有关 WebAssembly 的全面概述以及未来社区协作的路线图，请参阅[网络组装里程碑](https://hacks.mozilla.org/2016/03/a-webassembly-milestone/)在 Mozilla Hacks 博客上。*

自2015年6月以来，来自Google，Mozilla，Microsoft，Apple和[W3C WebAssembly Community Group](https://www.w3.org/community/webassembly/participants)一直在努力工作[设计](https://github.com/WebAssembly/design),[指定](https://github.com/WebAssembly/spec)，并实现 （[1](https://www.chromestatus.com/features/5453022515691520),[2](https://platform-status.mozilla.org/#web-assembly),[3](https://github.com/Microsoft/ChakraCore/wiki/Roadmap),[4](https://webkit.org/status/#specification-webassembly)WebAssembly，Web的新运行时和编译目标。[WebAssembly](https://webassembly.github.io/)是一种低级可移植字节码，设计为以紧凑的二进制格式进行编码，并在内存安全的沙箱中以接近本机的速度执行。作为现有技术的演变，WebAssembly与Web平台紧密集成，并且通过网络下载更快，实例化速度更快[asm.js](http://asmjs.org/)，JavaScript 的一个低级子集。

从今天开始，对WebAssembly的实验性支持在V8和Chromium中可用。要在 V8 中试用，请运行`d8`版本 5.1.117 或更高版本，从命令行使用`--expose_wasm`标记或打开“实验性 WebAssembly”功能`chrome://flags#enable-webassembly`在 Chrome Canary 51.0.2677.0 或更高版本中。重新启动浏览器后，新的`Wasm`对象将从JavaScript上下文中获得，该上下文公开了一个可以实例化和运行WebAssembly模块的API。**由于Mozilla和Microsoft的合作者的努力，WebAssembly的两个兼容实现也在一个标志后面运行。[火狐之夜](https://hacks.mozilla.org/2016/03/a-webassembly-milestone)并在内部构建中[微软边缘](http://blogs.windows.com/msedgedev/2016/03/15/previewing-webassembly-experiments)（在视频屏幕截图中演示）。**

WebAssembly项目网站有一个[演示](https://webassembly.github.io/demo/)展示运行时在 3D 游戏中的使用情况。在支持WebAssembly的浏览器中，演示页面将加载并实例化使用WebGL和其他Web平台API来呈现交互式游戏的wamm模块。在其他浏览器中，演示页面回退到同一游戏的 asm.js 版本。

![WebAssembly demo](/\_img/webassembly-experimental/tanks.jpg)

在引擎盖下，V8 中的 WebAssembly 实现旨在重用大部分现有的 JavaScript 虚拟机基础架构，特别是[涡轮风扇编译器](/blog/turbofan-jit).专用的 WebAssembly 解码器通过在一次传递中检查类型、局部变量索引、函数引用、返回值和控制流结构来验证模块。解码器生成一个TurboFan图，该图由各种优化过程处理，最后由同一后端转换为机器代码，该后端生成用于优化JavaScript和asm.js的机器代码。在接下来的几个月中，该团队将专注于通过编译器调优、并行性和编译策略改进来缩短 V8 实现的启动时间。

即将到来的两项更改也将显著改善开发人员体验。WebAssembly的标准文本表示将使开发人员能够像查看任何其他Web脚本或资源一样查看WebAssembly二进制文件的源代码。此外，当前占位符`Wasm`对象将被重新设计，以提供一组更强大，惯用的方法和属性，以实例化和内省JavaScript中的WebAssembly模块。

V8 / WebAssembly团队期待与其他浏览器供应商和更大的社区继续合作，因为我们正在努力实现运行时的稳定版本。我们还在计划[前途](https://github.com/WebAssembly/design/blob/master/PostMVP.md)WebAssembly 功能（包括[多线程](https://github.com/WebAssembly/design/blob/master/PostMVP.md#threads),[动态链接](https://github.com/WebAssembly/design/blob/master/DynamicLinking.md)和[GC / 一流的 DOM 集成](https://github.com/WebAssembly/design/blob/master/GC.md)），并继续开发工具链，用于编译 C、C++ 和其他语言[WebAssembly LLVM backend](http://llvm.org/docs/doxygen/html/WebAssembly\_8h.html)和[Emscripten](https://github.com/kripken/emscripten/wiki/WebAssembly).随着设计和实施过程的继续，请返回查看更多更新。
