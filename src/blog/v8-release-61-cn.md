***

标题： 'V8 版本 v6.1'
作者： 'V8团队'
日期： 2017-08-03 13：33：37
标签：

*   释放
    描述： 'V8 v6.1 附带了减小的二进制大小，并包括性能改进。此外，asm.js现在已经过验证并编译为WebAssembly。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.1](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.1)，直到几周后与Chrome 61 Stable合作发布之前，它一直处于测试阶段。V8 v6.1 充满了各种面向开发人员的好东西。我们想在发布之前预览一些亮点。

## 性能改进

访问地图和布景的所有元素 - 通过[迭 代](http://exploringjs.com/es6/ch_iteration.html)或[`Map.prototype.forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/forEach)/[`Set.prototype.forEach`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set/forEach)方法 — 速度明显加快，自 V8 版本 6.0 以来，原始性能提高了 11×检查[专门的博客文章](https://benediktmeurer.de/2017/07/14/faster-collection-iterators/)以获取更多信息。

![](/\_img/v8-release-61/iterating-collections.svg)

除此之外，其他语言功能的性能工作仍在继续。例如，[`Object.prototype.isPrototypeOf`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/isPrototypeOf)方法，这对于主要使用对象文本的无构造函数代码非常重要，并且`Object.create`而不是类和构造函数，现在总是像使用一样快，而且通常更快[这`instanceof`算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof).

![](/\_img/v8-release-61/checking-prototype.svg)

具有可变数量参数的函数调用和构造函数调用也明显更快。拨打电话[`Reflect.apply`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/apply)和[`Reflect.construct`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/construct)在最新版本中获得了高达17×的性能提升。

![](/\_img/v8-release-61/call-construct.svg)

`Array.prototype.forEach`现在已内联在 TurboFan 中，并针对所有主要无孔进行优化[元素种类](/blog/elements-kinds).

## 二进制大小减小

V8 团队完全删除了已弃用的 Crankshaft 编译器，从而显著减小了二进制大小。除了删除内置生成器之外，这还将 V8 的已部署二进制大小减少了 700 KB 以上，具体取决于确切的平台。

## asm.js现在已验证并编译为WebAssembly

如果 V8 遇到 asm.js代码，它现在会尝试验证它。然后，有效的 asm.js 代码被转译为 WebAssembly。根据 V8 的性能评估，这通常会提高吞吐量性能。由于添加了验证步骤，可能会发生启动性能的孤立回归。

请注意，默认情况下，此功能仅在铬侧处于打开状态。如果您是嵌入者并希望利用 asm.js 验证程序，请启用该标志`--validate-asm`.

## WebAssembly

调试 WebAssembly 时，现在可以在 WebAssembly 代码中的断点命中时在 DevTools 中显示局部变量。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.1 -t branch-heads/6.1`以试验 V8 v6.1 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
