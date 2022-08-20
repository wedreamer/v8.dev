***

标题： 'V8 发布 v5.4'
作者： 'V8团队'
日期： 2016-09-09 13：33：37
标签：

*   释放
    描述： “V8 v5.4 具有性能改进和内存消耗降低的功能。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.4)，它将处于测试阶段，直到几周后与Chrome 54 Stable协同发布。V8 v5.4 充满了各种面向开发人员的好东西，因此我们想在发布之前预览一些亮点。

## 性能改进

V8 v5.4 在内存占用和启动速度方面提供了许多关键改进。这些主要有助于加快初始脚本执行速度，并减少 Chrome 中的网页加载量。

### 记忆

在测量 V8 的内存消耗时，监控和了解两个指标非常重要：*峰值内存*消耗和*平均内存*消费。通常，降低峰值消耗与降低平均消耗同样重要，因为执行脚本即使在短时间内也会耗尽可用内存，这可能会导致*内存不足*崩溃，即使它的平均内存消耗不是很高。出于优化目的，将 V8 的内存分为两类很有用：*堆上内存*包含实际的 JavaScript 对象和*堆外内存*包含其余部分，例如编译器、解析器和垃圾回收器分配的内部数据结构。

在 5.4 中，我们为 RAM 为 512 MB 或更低的低内存设备调整了 V8 的垃圾回收器。根据显示的网站，这可以减少*峰值内存*消耗*堆上内存*为止**40%**.

V8 的 JavaScript 解析器中的内存管理得到了简化，以避免不必要的分配，从而减少了*堆外峰值内存*使用率最高可达**20%**.这些内存节省对于减少大型脚本文件（包括 asm.js 应用程序）的内存使用量特别有用。

### 启动和速度

我们简化 V8 解析器的工作不仅有助于减少内存消耗，还提高了解析器的运行时性能。这种简化，结合JavaScript内置的其他优化，以及JavaScript对象上属性的访问如何使用全局[内联缓存](https://en.wikipedia.org/wiki/Inline_caching)，从而显著提高了启动性能。

我们[内部启动测试套件](https://www.youtube.com/watch?v=xCx4uC7mn6Y)衡量实际JavaScript性能的中位数提高了5%。这[速度计](http://browserbench.org/Speedometer/)基准测试也受益于这些优化，通过以下方式进行改进[~10% 至 13%，而 v5.2](https://chromeperf.appspot.com/report?sid=f5414b72e864ffaa4fd4291fa74bf3fd7708118ba534187d36113d8af5772c86\&start_rev=393766\&end_rev=416239).

![](/\_img/v8-release-54/speedometer.png)

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.4 -t branch-heads/5.4`以试验 V8 v5.4 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
