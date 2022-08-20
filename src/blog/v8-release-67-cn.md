***

标题： 'V8 版本 v6.7'
作者： 'V8团队'
日期： 2018-05-04 13：33：37
标签：

*   释放
    推文：“992506342391742465”
    描述：“V8 v6.7 添加了更多不受信任的代码缓解措施，并提供了 BigInt 支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.7)，直到几周后与Chrome 67 Stable合作发布之前，它一直处于测试阶段。V8 v6.7 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript 语言特性

V8 v6.7 附带了默认启用的 BigInt 支持。BigInts 是 JavaScript 中一种新的数字基元，可以任意精度表示整数。读[我们的 BigInt 功能解释器](/features/bigint)有关如何在 JavaScript 中使用 BigInts 的更多信息，请查看[我们写了更多关于V8实现的细节](/blog/bigint).

## 不受信任的代码缓解措施

在 V8 v6.7 中，我们登陆[针对侧信道漏洞的更多缓解措施](/docs/untrusted-code-mitigations)以防止信息泄露给不受信任的 JavaScript 和 WebAssembly 代码。

## V8 接口

请使用`git log branch-heads/6.6..branch-heads/6.7 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.7 -t branch-heads/6.7`以试验 V8 v6.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
