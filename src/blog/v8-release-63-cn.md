***

标题： 'V8 版本 v6.3'
作者： 'V8团队'
日期： 2017-10-25 13：33：37
标签：

*   释放
    描述： “V8 v6.3 包括性能改进、减少内存消耗以及对新的 JavaScript 语言功能的支持。
    推文：“923168001108643840”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.3](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.3)，直到几周后与Chrome 63 Stable一起发布之前，它一直处于测试阶段。V8 v6.3 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 速度

[詹克·巴斯特斯](/blog/jank-busters)三、上架作为一部分[奥里诺科河](/blog/orinoco)项目。并发标记（[70-80%](https://chromeperf.appspot.com/report?sid=612eec65c6f5c17528f9533349bad7b6f0020dba595d553b1ea6d7e7dcce9984)的标记是在非阻塞线程上完成的）发货。

解析器现在不[需要第二次准备函数](https://docs.google.com/document/d/1TqpdGeLmURL2gc18s6PwNeyZOvayQJtJ16TCn0BEt48/edit#heading=h.un2pnqwbiw11).这转化为[解析时间中位数提高 14%](https://docs.google.com/document/d/1TqpdGeLmURL2gc18s6PwNeyZOvayQJtJ16TCn0BEt48/edit#heading=h.dvuo4tqnsmml)在我们的内部创业公司Top25基准测试中。

`string.js`已完全移植到 CodeStubAssembler。非常感谢[@peterwmwong](https://twitter.com/peterwmwong)为[他令人敬畏的贡献](https://chromium-review.googlesource.com/q/peter.wm.wong)!作为开发人员，这意味着内置字符串的功能如下`String#trim`从 V8 v6.3 开始，速度要快得多。

`Object.is()`现在的表现与替代方案大致相当。总的来说，V8 v6.3 继续朝着更好的 ES2015+ 性能迈进。除了其他项目，我们提升了[多态访问符号的速度](https://bugs.chromium.org/p/v8/issues/detail?id=6367),[构造函数调用的多态内联](https://bugs.chromium.org/p/v8/issues/detail?id=6885)和[（标记）模板文本](https://pasteboard.co/GLYc4gt.png).

![V8’s performance over the past six releases](/\_img/v8-release-63/ares6.svg)

弱优化功能列表消失了。更多信息可以在[专门的博客文章](/blog/lazy-unlinking).

上述项目是速度改进的非详尽列表。还发生了许多其他与性能相关的工作。

## 内存消耗

[写入障碍切换到使用 CodeStubAssembler](https://chromium.googlesource.com/v8/v8/+/dbfdd4f9e9741df0a541afdd7516a34304102ee8).这样可以为每个隔离节省大约 100 KB 的内存。

## JavaScript 语言特性

V8 现在支持以下阶段 3 功能：[动态模块导入方式`import()`](/features/dynamic-import),[`Promise.prototype.finally()`](/features/promise-finally)和[异步迭代器/生成器](https://github.com/tc39/proposal-async-iteration).

跟[动态模块导入](/features/dynamic-import)根据运行时条件导入模块非常简单。当应用程序应该延迟加载某些代码模块时，这会派上用场。

[`Promise.prototype.finally`](/features/promise-finally)引入了一种在承诺确定后轻松清理的方法。

随着异步函数的引入，使用异步函数进行迭代变得更加符合人体工程学[异步迭代器/生成器](https://github.com/tc39/proposal-async-iteration).

在`Intl`边[`Intl.PluralRules`](/features/intl-pluralrules)现在受支持。此 API 支持高性能的国际化复数。

## 检查器/调试

在 Chrome 63 中[块覆盖](https://docs.google.com/presentation/d/1IFqqlQwJ0of3NuMvcOk-x4P_fpi1vJjnjGrhQCaJkH4/edit#slide=id.g271d6301ff\_0\_44)在 DevTools UI 中也受支持。请注意，自 V8 v6.2 起，检查器协议已经支持块覆盖。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用 git checkout -b 6.3 -t branch-heads/6.3 来试验 V8 v6.3 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
