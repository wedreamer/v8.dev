***

标题： 'V8 发布 v8.9'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），正在等待呼叫'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-02-04
    标签：
*   释放
    描述： “V8 版本 v8.9 为参数大小不匹配的调用带来了性能改进。
    推文：“1357358418902802434”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.9)，直到几周后与Chrome 89 Stable合作发布之前，它一直处于测试阶段。V8 v8.9 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 顶级`await`

[顶级`await`](https://v8.dev/features/top-level-await)在[眨眼渲染引擎](https://www.chromium.org/blink)89，V8的主嵌入器。

在独立 V8 中，顶级`await`仍然在后面`--harmony-top-level-await`旗。

请参阅[我们的解释员](https://v8.dev/features/top-level-await)了解更多详情。

## 性能

### 参数大小不匹配的更快调用

JavaScript允许调用参数数与预期参数数不同的函数，即可以传递比声明的形式参数更少或更多的参数。前一种情况称为“应用不足”，后者称为“过度应用”。

在应用不足的情况下，其余参数将分配给`undefined`价值。在过度应用的情况下，可以使用 rest 参数和`Function.prototype.arguments`财产，或者它们只是多余的和被忽视的。如今，许多Web和Node.js框架都使用此JS功能来接受可选参数并创建更灵活的API。

直到最近，V8才有一个特殊的机制来处理参数大小不匹配：参数适配器框架。不幸的是，参数适应是以性能为代价的，并且在现代前端和中间件框架中通常需要。事实证明，通过巧妙的设计（例如反转堆栈中参数的顺序），我们可以删除这个额外的帧，简化V8代码库，并几乎完全消除开销。

![Performance impact of removing the arguments adaptor frame, as measured through a micro-benchmark.](../_img/v8-release-89/perf.svg)

该图显示，在 上运行时不再有开销[无 JIT 模式](https://v8.dev/blog/jitless)（点火）性能提高11.2%。使用 TurboFan 时，我们的加速速度提高了 40%。与无不匹配情况相比，开销是由于[函数尾声](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/backend/x64/code-generator-x64.cc;l=4905;drc=5056f555010448570f7722708aafa4e55e1ad052).有关更多详细信息，请参阅[设计文档](https://docs.google.com/document/d/15SQV4xOhD3K0omGJKM-Nn8QEaskH7Ir1VYJb9\_5SjuM/edit).

如果您想了解有关这些改进背后的详细信息的更多信息，请查看[专门的博客文章](https://v8.dev/blog/adaptor-frame).

## V8 接口

请使用`git log branch-heads/8.8..branch-heads/8.9 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.9 -t branch-heads/8.9`以试验 V8 v8.9 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
