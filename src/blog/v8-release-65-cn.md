***

标题： 'V8 版本 v6.5'
作者： 'V8团队'
日期： 2018-02-01 13：33：37
标签：

*   释放
    描述： “V8 v6.5 增加了对流式处理 WebASsembly 编译的支持，并包括一个新的”不受信任的代码模式”。
    推文：“959174292406640640”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.5)，直到几周后与Chrome 65 Stable一起发布之前，它一直处于测试阶段。V8 v6.5 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 不受信任的代码模式

为了应对最新的投机性侧信道攻击，名为Spectre，V8引入了一个[不受信任的代码模式](/docs/untrusted-code-mitigations).如果嵌入 V8，请考虑利用此模式，以防应用程序处理用户生成的不可信代码。请注意，该模式在默认情况下处于启用状态，包括在 Chrome 中。

## WebAssembly 代码的流编译

WebAssembly API 提供了一个特殊的函数来支持[流式编译](https://developers.google.com/web/updates/2018/04/loading-wasm)与`fetch()`应用程序接口：

```js
const module = await WebAssembly.compileStreaming(fetch('foo.wasm'));
```

此 API 自 V8 v6.1 和 Chrome 61 起就可用，尽管最初的实现实际上并未使用流编译。但是，在 V8 v6.5 和 Chrome 65 中，我们利用此 API 并编译 WebAssembly 模块，同时仍在下载模块字节。一旦我们下载单个函数的所有字节，我们就会将该函数传递给后台线程进行编译。

我们的测量表明，使用此API，Chrome 65中的WebAssembly编译可以在高端计算机上跟上高达50 Mbit / s的下载速度。这意味着，如果您以 50 Mbit/s 的速度下载 WebAssembly 代码，则该代码的编译将在下载完成后立即完成。

对于下图，我们测量了下载和编译具有67 MB和大约190，000个函数的WebAssembly模块所需的时间。我们以25 Mbit /s，50 Mbit / s和100 Mbit / s的下载速度进行测量。

![](../_img/v8-release-65/wasm-streaming-compilation.svg)

当下载时间长于WebAssembly模块的编译时间时，例如，在上图中，25 Mbit/s和50 Mbit/s，则`WebAssembly.compileStreaming()`在下载最后一个字节后几乎立即完成编译。

当下载时间短于编译时间时，则`WebAssembly.compileStreaming()`编译 WebAssembly 模块所需的时间与编译 WebAssembly 模块所需的时间一样长，而无需先下载模块。

## 速度

我们继续致力于扩大JavaScript内置的快速路径，添加一种机制来检测和防止毁灭性的情况，称为“去优化循环”。当您的优化代码取消优化时，就会发生这种情况，并且存在*没有办法知道哪里出了问题*.在这种情况下，TurboFan只是不断尝试优化，最终在大约30次尝试后放弃了。如果您在任何二阶数组内置的回调函数中执行了一些操作来更改数组的形状，则会发生这种情况。例如，更改`length`数组 — 在 V8 v6.5 中，我们会注意到这种情况何时发生，并停止在将来的优化尝试中内联在该站点调用的数组内置。

我们还通过内联许多以前由于要调用的函数的负载与调用本身（例如函数调用）之间的副作用而被排除的内置函数来扩大快速路径。和`String.prototype.indexOf`得到一个[10× 函数调用性能改进](https://bugs.chromium.org/p/v8/issues/detail?id=6270).

在 V8 v6.4 中，我们内联了对`Array.prototype.forEach`,`Array.prototype.map`和`Array.prototype.filter`.在 V8 v6.5 中，我们添加了对以下各项的内联支持：

*   `Array.prototype.reduce`
*   `Array.prototype.reduceRight`
*   `Array.prototype.find`
*   `Array.prototype.findIndex`
*   `Array.prototype.some`
*   `Array.prototype.every`

此外，我们已经拓宽了所有这些内置的快速路径。起初，我们会在看到带有浮点数的数组时进行救援，或者（甚至更多救援）[如果数组中有“孔”](/blog/elements-kinds)，例如`[3, 4.5, , 6]`.现在，我们处理除`find`和`findIndex`，其中规范要求将孔转换为`undefined`在我们的努力中投入了一个猴子扳手（*现在...！*).

下图显示了与 V8 v6.4 相比，我们的内联内置功能（细分为整数数组、双数组和带孔的双数组）的改进增量。时间以毫秒为单位。

![Performance improvements since V8 v6.4](../_img/v8-release-65/performance-improvements.svg)

## V8 接口

请使用`git log branch-heads/6.4..branch-heads/6.5 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.5 -t branch-heads/6.5`以试验 V8 v6.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
