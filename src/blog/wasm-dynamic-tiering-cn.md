***

标题： 'WebAssembly Dynamic Tiering 准备在 Chrome 96 中试用”
作者：“安德烈亚斯·哈斯 - 蒂里施的乐趣”
化身：

*   安德烈亚斯-哈斯
    日期： 2021-10-29
    标签：
*   WebAssembly
    描述： “WebAssembly Dynamic Tiering 准备在 V8 v9.6 和 Chrome 96 中试用，无论是通过命令行标志，还是通过源试用版”
    推文：“1454158971674271760”

***

V8 有两个编译器，用于将 WebAssembly 代码编译为机器代码，然后可以执行：基线编译器**起飞**和优化编译器**涡轮风扇**.Liftoff可以比TurboFan更快地生成代码，从而可以快速启动。另一方面，TurboFan可以生成更快的代码，从而实现高峰值性能。

在Chrome的当前配置中，WebAssembly模块首先完全由Liftoff编译。Liftoff编译完成后，整个模块会立即由TurboFan在后台再次编译。通过流编译，如果 Liftoff 编译 WebAssembly 代码的速度比下载 WebAssembly 代码的速度快，TurboFan 编译可以更早开始。初始 Liftoff 编译允许快速启动时间，而后台的 TurboFan 编译可尽快提供高峰值性能。有关Liftoff，TurboFan和整个编译过程的更多详细信息，可以在[单独的文档](https://v8.dev/docs/wasm-compilation-pipeline).

使用TurboFan编译整个WebAssembly模块可以在编译完成后提供最佳性能，但这是有代价的：

*   在后台执行 TurboFan 编译的 CPU 内核可以阻止需要 CPU 的其他任务，例如 Web 应用程序的工作线程。
*   不重要功能的 TurboFan 编译可能会延迟 TurboFan 对更重要功能的编译，从而延迟 Web 应用程序达到全部性能。
*   一些WebAssembly函数可能永远不会被执行，并且花费资源来编译这些函数与TurboFan可能不值得。

## 动态分层

动态分层应该通过仅使用TurboFan编译那些实际执行多次的函数来缓解这些问题。因此，动态分层可以通过多种方式改变 Web 应用程序的性能：动态分层可以通过减少 CPU 上的负载来加快启动时间，从而允许 WebAssembly 编译以外的启动任务更多地使用 CPU。动态分层还会延迟重要功能的 TurboFan 编译，从而降低性能。例如，由于 V8 不对 WebAssembly 代码使用堆栈上替换，因此执行可能会卡在 Liftoff 代码中的循环中。此外，代码缓存也会受到影响，因为Chrome仅缓存TurboFan代码，并且所有不符合TurboFan编译条件的函数都会在启动时使用Liftoff进行编译，即使编译的WebAssembly模块已经存在于缓存中也是如此。

## 如何尝试

我们鼓励感兴趣的开发人员尝试动态分层对其 Web 应用程序的性能影响。这将使我们能够做出反应，并尽早避免潜在的性能下降。可以通过运行带有命令行标志的 Chrome 在本地启用动态分层`--enable-blink-features=WebAssemblyDynamicTiering`.

想要启用动态分层的 V8 嵌入器可以通过设置 V8 标志来执行此操作`--wasm-dynamic-tiering`.

### 通过原产地试验进行现场测试

使用命令行标志运行 Chrome 是开发人员可以执行的操作，但最终用户不应期望这样做。要在现场试验您的应用程序，可以加入所谓的[原产地试用](https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md).源试用版允许您通过绑定到域的特殊令牌与最终用户一起试用实验性功能。此特殊令牌在包含令牌的特定页面上为最终用户启用 WebAssembly 动态分层。要获取您自己的令牌以运行源试用版，[使用申请表](https://developer.chrome.com/origintrials/#/view_trial/3716595592487501825).

## 给我们反馈

我们正在寻找尝试此功能的开发人员的反馈，因为它将有助于在TurboFan编译有用时以及TurboFan编译没有回报并且可以避免时获得正确的启发式方法。发送反馈的最佳方式是[报告问题](https://bugs.chromium.org/p/chromium/issues/detail?id=1260322).
