***

标题： 'V8 版本 v5.3'
作者： 'V8团队'
日期： 2016-07-18 13：33：37
标签：

*   释放
    描述： “V8 v5.3 带来了性能改进和内存消耗的降低。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.3](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.3)，它将处于测试阶段，直到与Chrome 53 Stable协同发布。V8 v5.3 充满了各种面向开发人员的好东西，所以我们想在几周后发布之前预览一些亮点。

## 记忆

### 新的点火解释器

Ignition是V8的新解释器，功能齐全，将在Chrome 53中为低内存Android设备启用。该解释器可立即为JIT代码节省内存，并允许V8进行未来优化，以便在代码执行期间更快地启动。Ignition与V8现有的优化编译器（TurboFan和曲轴）协同工作，以确保“热”代码仍然针对峰值性能进行优化。我们将继续提高口译员的性能，并希望尽快在所有平台，移动和桌面上启用Ignition。查找即将发布的博客文章，了解有关 Ignition 设计、体系结构和性能提升的更多信息。V8的嵌入式版本可以打开带有标志的点火解释器`--ignition`.

### 减少卡顿

V8 v5.3 包含各种更改，以减少应用程序卡顿和垃圾回收时间。这些更改包括：

*   优化弱全局句柄，以减少处理外部存储器所花费的时间
*   统一堆以进行完整的垃圾回收，以减少疏散卡顿
*   优化 V8 的[黑色分配](/blog/orinoco)垃圾回收标记阶段的新增内容

总之，这些改进将完整垃圾回收暂停时间减少了约 25%，这是在浏览热门网页语料库时测量的。有关最近为减少卡顿而进行的垃圾回收优化的更多详细信息，请参阅“Jank Busters”博客文章[第 1 部分](/blog/jank-busters)&[第 2 部分](/blog/orinoco).

## 性能

### 缩短页面启动时间

V8团队最近开始根据25个现实世界的网站页面加载（包括Facebook，Reddit，Wikipedia和Instagram等热门网站）来跟踪性能改进。在 V8 v5.1（从 4 月开始在 Chrome 51 中测量）和 V8 v5.3（在最近的 Chrome Canary 53 中测量）之间，我们测量的网站的启动时间总体上缩短了约 7%。加载真实网站的这些改进反映了Speedometer基准测试的类似收益，该基准测试在V8 v5.3中的运行速度提高了14%。有关我们新的测试工具、运行时改进以及 V8 在页面加载期间花费时间的细分分析的更多详细信息，请参阅我们即将发布的有关启动性能的博客文章。

### ES2015`Promise`性能

V8在[蓝鸟 ES2015`Promise`基准测试套件](https://github.com/petkaantonov/bluebird/tree/master/benchmark)在 V8 v5.3 中改进了 20–40%，因体系结构和基准测试而异。

![V8’s Promise performance over time on a Nexus 5x](../_img/v8-release-53/promise.png)

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 5.3 -t branch-heads/5.3`以试验 V8 5.3 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
