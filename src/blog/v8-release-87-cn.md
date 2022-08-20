***

标题： 'V8 发布 v8.7'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），V8旗手
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2020-10-23
    标签：
*   释放
    描述： 'V8 release v8.7 带来了用于本机调用的新 API，Atomics.waitAsync，错误修复和性能改进。
    推文：“1319654229863182338”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.7)，直到几周后与Chrome 87 Stable一起发布，它一直处于测试阶段。V8 v8.7 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 不安全的快速JS调用

V8 v8.7 附带了一个增强的 API，用于从 JavaScript 执行本机调用。

该功能仍处于实验阶段，可以通过`--turbo-fast-api-calls`标志在 V8 或相应的`--enable-unsafe-fast-js-calls`标记。它旨在提高Chrome中某些本机图形API的性能，但也可以由其他嵌入器使用。它为开发人员提供了创建实例的新方法`v8::FunctionTemplate`，如本文所述[头文件](https://source.chromium.org/chromium/chromium/src/+/master:v8/include/v8-fast-api-calls.h).使用原始 API 创建的函数将不受影响。

有关详细信息和可用功能列表，请参阅[这个解释器](https://docs.google.com/document/d/1nK6oW11arlRb7AA76lJqrBIygqjgdc92aXUPYecc9dU/edit?usp=sharing).

### `Atomics.waitAsync`

[`Atomics.waitAsync`](https://github.com/tc39/proposal-atomics-wait-async/blob/master/PROPOSAL.md)现已在 V8 v8.7 中可用。

[`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)和[`Atomics.notify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/notify)是低级同步基元，可用于实现互斥锁和其他同步方式。但是，由于`Atomics.wait`是阻塞的，不可能在主线程上调用它（尝试这样做会抛出一个TypeError）。非阻塞版本，[`Atomics.waitAsync`](https://github.com/tc39/proposal-atomics-wait-async/blob/master/PROPOSAL.md)，也可以在主线程上使用。

退房[我们的解释器在`Atomics`蜜蜂属](https://v8.dev/features/atomics)了解更多详情。

## V8 接口

请使用`git log branch-heads/8.6..branch-heads/8.7 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.7 -t branch-heads/8.7`以试验 V8 v8.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
