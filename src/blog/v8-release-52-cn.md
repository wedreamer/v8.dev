***

标题： 'V8 版本 v5.2'
作者： 'V8团队'
日期： 2016-06-04 13：33：37
标签：

*   释放
    描述：“V8 v5.2 包括对 ES2016 语言功能的支持。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.2](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.2)，它将处于测试阶段，直到与Chrome 52 Stable协同发布。V8 5.2 充满了各种面向开发人员的好东西，所以我们想在几周后发布之前预览一些亮点。

## ES2015 和 ES2016 支持

V8 v5.2 包含对 ES2015（又名 ES6）和 ES2016（又名 ES7）的支持。

### 幂运算符

此版本包含对 ES2016 幂运算符的支持，这是一种要替换的中缀表示法`Math.pow`.

```js
let n = 3**3; // n == 27
n **= 2; // n == 729
```

### 不断发展的规格

有关支持不断发展的规范背后的复杂性以及围绕 Web 兼容性 bug 和尾部调用的持续标准讨论的更多信息，请参阅 V8 博客文章[ES2015、ES2016 及以后版本](/blog/modern-javascript).

## 性能

V8 v5.2 包含进一步的优化，以提高 JavaScript 内置程序的性能，包括对 array 操作的改进，如 isArray 方法、in 运算符和 Function.prototype.bind。这是正在进行的工作的一部分，旨在根据对流行网页上的运行时调用统计信息的新分析来加速内置功能。有关详细信息，请参阅[V8 谷歌 I/O 2016 讲座](https://www.youtube.com/watch?v=N1swY14jiKc)并查找即将发布的有关从实际网站收集的性能优化的博客文章。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 5.2 -t branch-heads/5.2`以试验 V8 v5.2 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
