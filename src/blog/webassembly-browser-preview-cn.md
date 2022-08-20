***

标题： 'WebAssembly 浏览器预览'
作者： 'V8团队'
日期： 2016-10-31 13：33：37
标签：

*   WebAssembly
    描述： 'WebAssembly或Wasm是Web的新运行时和编译目标，现在可以在Chrome Canary的标志后面使用！

***

今天，我们很高兴地宣布，与[火狐浏览器](https://hacks.mozilla.org/2016/10/webassembly-browser-preview)和[边缘](https://blogs.windows.com/msedgedev/2016/10/31/webassembly-browser-preview/)，WebAssembly 浏览器预览。[WebAssembly](http://webassembly.org/)或者Wasm是Web的新运行时和编译目标，由Google，Mozilla，Microsoft，Apple和[W3C WebAssembly Community Group](https://www.w3.org/community/webassembly/).

## 这个里程碑标志着什么？

这个里程碑意义重大，因为它标志着：

*   我们的候选版本[最有价值球员](http://webassembly.org/docs/mvp/)（最小可行产品）设计（包括[语义学](http://webassembly.org/docs/semantics/),[二进制格式](http://webassembly.org/docs/binary-encoding/)和[JS API](http://webassembly.org/docs/js/))
*   WebAssembly的兼容和稳定的实现在V8和SpiderMonkey的树干标志后面，在Chakra的开发版本中，以及在JavaScriptCore中进行中
*   一个[工作工具链](http://webassembly.org/getting-started/developers-guide/)供开发人员从 C/C++ 源文件编译 WebAssembly 模块
*   一个[路线图](http://webassembly.org/roadmap/)默认发布 WebAssembly，禁止根据社区反馈进行更改

您可以在[项目网站](http://webassembly.org/)以及关注我们的[开发人员指南](http://webassembly.org/getting-started/developers-guide/)使用Emscripten测试C\&C++的WebAssembly编译。这[二进制格式](http://webassembly.org/docs/binary-encoding/)和[JS API](http://webassembly.org/docs/js/)文档分别概述了WebAssembly的二进制编码以及在浏览器中实例化WebAssembly模块的机制。下面是一个快速示例，用于显示 wasm 的外观：

![An implementation of the Greatest Common Divisor function in WebAssembly, showing the raw bytes, the text format (WAST), and the C source code.](/\_img/webassembly-browser-preview/gcd.svg)

由于WebAssembly仍然在Chrome中的标志后面（<chrome://flags/#enable-webassembly>），目前不建议将其用于生产用途。但是，浏览器预览期间标记了我们主动收集的时间[反馈](http://webassembly.org/community/feedback/)关于规范的设计和实现。鼓励开发人员测试编译和移植应用程序并在浏览器中运行它们。

V8 继续优化 WebAssembly 在[涡轮风扇编译器](/blog/turbofan-jit).自去年三月我们首次宣布实验性支持以来，我们增加了对并行编译的支持。此外，我们即将完成一个替代的asm.js管道，它将asm.js转换为WebAssembly。[在引擎盖下](https://www.chromestatus.com/feature/5053365658583040)因此，现有的 asm.js 站点可以获得 WebAssembly 提前编译的一些好处。

## 下一步是什么？

除非社区反馈导致重大设计变更，否则WebAssembly社区小组计划在2017年第一季度发布官方规范，届时将鼓励浏览器默认发布WebAssembly。从那时起，二进制格式将被重置为版本1，WebAssembly将是无版本，功能测试和向后兼容。更详细[路线图](http://webassembly.org/roadmap/)可以在WebAssembly项目网站上找到。
