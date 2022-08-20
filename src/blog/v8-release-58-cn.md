***

标题： 'V8 版本 v5.8'
作者： 'V8团队'
日期： 2017-03-20 13：33：37
标签：

*   释放
    描述： “V8 v5.8 支持使用任意堆大小并提高启动性能。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.8](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.8)，它将处于测试阶段，直到几周后与Chrome 58 Stable协同发布。V8 5.8 充满了各种面向开发人员的好东西。我们想在发布之前预览一些亮点。

## 任意堆大小

从历史上看，V8 堆限制被方便地设置为适合有符号的 32 位整数范围，并带有一些边距。随着时间的推移，这种便利性导致V8中的代码草率，混合了不同位宽度的类型，有效地打破了增加限制的能力。在 V8 v5.8 中，我们启用了任意堆大小的使用。查看[专门的博客文章](/blog/heap-size-limit)了解更多信息。

## 启动性能

在 V8 v5.8 中，我们继续致力于逐步减少启动期间在 V8 中花费的时间。编译和解析代码所花费的时间减少，以及 IC 系统的优化，使我们的[实际的启动工作负载](/blog/real-world-performance).

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.8 -t branch-heads/5.8`以试验 V8 5.8 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
