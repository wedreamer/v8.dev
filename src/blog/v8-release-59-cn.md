***

标题： 'V8 版本 v5.9'
作者： 'V8团队'
日期： 2017-04-27 13：33：37
标签：

*   释放
    描述： 'V8 v5.9 包括新的 Ignition + TurboFan 流水线，并在所有平台上添加了 WebAssembly TrapIf 支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.9)，它将处于测试阶段，直到几周后与Chrome 59 Stable协同发布。V8 5.9 充满了各种面向开发人员的好东西。我们想在发布之前预览一些亮点。

## 点火+涡轮风扇发射

V8 v5.9将是第一个默认启用Ignition+TurboFan的版本。通常，此开关应该可以降低 Web 应用程序的内存消耗和更快的启动速度，并且我们预计不会出现稳定性或性能问题，因为新管道已经过重大测试。然而[给我们打电话](https://bugs.chromium.org/p/v8/issues/entry?template=Bug%20report%20for%20the%20new%20pipeline)以防您的代码突然开始性能显著下降。

有关详细信息，请参阅[我们的专用博客文章](/blog/launching-ignition-and-turbofan).

## WebAssembly`TrapIf`支持所有平台

[WebAssembly`TrapIf`支持](https://chromium.googlesource.com/v8/v8/+/98fa962e5f342878109c26fd7190573082ac3abe)显著减少了编译代码所花费的时间（约 30%）。

![](../_img/v8-release-59/angrybots.png)

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.9 -t branch-heads/5.9`以试验 V8 5.9 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
