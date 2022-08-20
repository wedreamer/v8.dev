***

标题： 'V8 版本 v9.6'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-10-13
    标签：
*   释放
    描述： 'V8 版本 v9.6 为 WebAssembly 带来了对引用类型的支持。
    推文：“1448262079476076548”

***

每四周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.6)，直到几周后与Chrome 96 Stable一起发布之前，它一直处于测试阶段。V8 v9.6 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## WebAssembly

### 引用类型

这[引用类型建议](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md)，在 V8 v9.6 中提供，允许在 WebAssembly 模块中不透明地使用来自 JavaScript 的外部引用。这`externref`（以前称为`anyref`） 数据类型提供了一种保存对 JavaScript 对象的引用的安全方法，并与 V8 的垃圾回收器完全集成。

已经对参考类型具有可选支持的工具链很少[wasm-bindgen for Rust](https://rustwasm.github.io/wasm-bindgen/reference/reference-types.html)和[汇编脚本](https://www.assemblyscript.org/compiler.html#command-line-options).

## V8 接口

请使用`git log branch-heads/9.5..branch-heads/9.6 include/v8\*.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.6 -t branch-heads/9.6`以试验 V8 v9.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
