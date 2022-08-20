***

标题： “发射点火和涡轮风扇”
作者： 'V8团队'
日期： 2017-05-15 13：33：37
标签：

*   内部
    描述： 'V8 v5.9 带有一个全新的 JavaScript 执行流水线，建立在 Ignition 解释器和 TurboFan 优化编译器之上。

***

今天，我们很高兴地宣布推出适用于 V8 v5.9 的新 JavaScript 执行管道，该管道将在 v59 中实现 Chrome Stable。通过新的管道，我们在实际的JavaScript应用程序上实现了巨大的性能改进和显着的内存节省。我们将在本文末尾更详细地讨论这些数字，但首先让我们看一下管道本身。

新管道建立在[点火](/docs/ignition)、V8 的解释器，以及[涡轮风扇](/docs/turbofan)，V8 最新的优化编译器。这些技术[应该](/blog/turbofan-jit) [是](/blog/ignition-interpreter) [熟悉](/blog/test-the-future)对于那些在过去几年中关注V8博客的人来说，但是切换到新的管道标志着两者的一个重要的新里程碑。

<figure>
  <img src="/_img/v8-ignition.svg" width="256" height="256" alt="" loading="lazy">
  <figcaption>Logo for Ignition, V8’s brand-new interpreter</figcaption>
</figure>

<figure>
  <img src="/_img/v8-turbofan.svg" width="256" height="256" alt="" loading="lazy">
  <figcaption>Logo for TurboFan, V8’s brand-new optimizing compiler</figcaption>
</figure>

Ignition 和 TurboFan 首次在 V8 v5.9 中被普遍和专门用于 JavaScript 执行。此外，从 v5.9 开始，全编码机和曲轴技术[自2010年以来一直为V8提供服务](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html)，在 V8 中不再用于 JavaScript 执行，因为它们不再能够跟上新的 JavaScript 语言特性以及这些特性所需的优化的步伐。我们计划很快完全删除它们。这意味着V8将拥有一个整体上更简单，更易于维护的架构。

## 漫长的旅程

结合Ignition和TurboFan管道已经开发了近3年半。它代表了 V8 团队从测量实际 JavaScript 性能并仔细考虑 Full-codegen 和 Crankshaft 的缺点中收集的集体见解的顶峰。这是一个基础，我们将能够在未来几年继续优化整个JavaScript语言。

TurboFan项目最初于2013年底启动，旨在解决曲轴的缺点。曲轴只能优化JavaScript语言的一个子集。例如，它不是为使用结构化异常处理来优化JavaScript代码而设计的，即由JavaScript的try，catch和最后的关键字划分的代码块。很难在 Crankshaft 中添加对新语言功能的支持，因为这些功能几乎总是需要为九个受支持的平台编写特定于体系结构的代码。此外，曲轴的架构在能够生成最佳机器代码方面受到限制。它只能从JavaScript中挤出如此多的性能，尽管要求V8团队维护每个芯片架构超过一万行代码。

TurboFan从一开始就被设计出来，不仅是为了优化当时JavaScript标准ES5中的所有语言功能，也是为了优化ES2015及以后计划的所有未来功能。它引入了分层编译器设计，可以在高级和低级编译器优化之间实现清晰的分离，从而可以轻松添加新的语言功能，而无需修改特定于体系结构的代码。TurboFan增加了一个明确的指令选择编译阶段，使得首先为每个支持的平台编写更少的特定于架构的代码成为可能。在这个新阶段，特定于体系结构的代码只需编写一次，并且很少需要更改。这些和其他决策为 V8 支持的所有架构带来了一个更易于维护和扩展的优化编译器。

V8的Ignition解释器背后的最初动机是减少移动设备上的内存消耗。在 Ignition 之前，V8 的 Full-codegen 基线编译器生成的代码通常占据了 Chrome 中整个 JavaScript 堆的近三分之一。这为 Web 应用程序的实际数据留出了更少的空间。当在RAM有限的Android设备上为Chrome M53启用Ignition时，在基于ARM64的移动设备上，基线，未优化的JavaScript代码所需的内存占用量减少了九倍。

后来，V8团队利用了Ignition的字节码可以用来直接使用TurboFan生成优化的机器代码，而不必像Crankshaft那样从源代码重新编译。Ignition的字节码在V8中提供了一个更干净，更不容易出错的基线执行模型，简化了去优化机制，这是V8的一个关键特性。[自适应优化](https://en.wikipedia.org/wiki/Adaptive_optimization).最后，由于生成字节码比生成 Full-codegen 的基线编译代码更快，因此激活 Ignition 通常会缩短脚本启动时间，进而缩短网页加载速度。

通过将Ignition和TurboFan的设计紧密结合，整体架构有更多的好处。例如，V8团队没有在手工编码的汇编中编写Ignition的高性能字节码处理程序，而是使用TurboFan的[中间表示](https://en.wikipedia.org/wiki/Intermediate_representation)来表达处理程序的功能，并让 TurboFan 为 V8 的众多受支持平台进行优化和最终代码生成。这确保了Ignition在V8所有支持的芯片架构上都能表现出色，同时消除了维护九个独立平台端口的负担。

## 运行数字

撇开历史不谈，现在让我们来看看新管道的实际性能和内存消耗。

V8 团队使用[遥测 - 弹射器](https://catapult.gsrc.io/telemetry)框架。[以前](/blog/real-world-performance)在这篇博客中，我们讨论了为什么使用来自实际测试的数据来推动我们的性能优化工作以及我们如何使用它如此重要。[网页回放](https://github.com/chromium/web-page-replay)与遥测一起执行此操作。切换到Ignition和TurboFan显示了这些实际测试用例的性能改进。具体来说，新的管道可以显著加快知名网站的用户交互故事测试速度：

![Reduction in time spent in V8 for user interaction benchmarks](/\_img/launching-ignition-and-turbofan/improvements-per-website.png)

虽然 Speedometer 是一个合成基准测试，但我们之前已经发现，与其他综合基准测试相比，它在近似现代 JavaScript 的实际工作负载方面做得更好。切换到点火和TurboFan可将V8的速度计分数提高5%-10%，具体取决于平台和设备。

新的管道还加快了服务器端JavaScript的速度。[阿克美尔](https://github.com/acmeair/acmeair-nodejs)，Node.js的基准测试，模拟虚构航空公司的服务器后端实现，使用V8 v5.9的运行速度提高了10%以上。

![Improvements on Web and Node.js benchmarks](/\_img/launching-ignition-and-turbofan/benchmark-scores.png)

Ignition和TurboFan还减少了V8的整体内存占用。在Chrome M59中，新的管道将V8在台式机和高端移动设备上的内存占用量减少了5-10%。这种减少是带来点火内存节省的结果，这些内存已经[以前涵盖的内容](/blog/ignition-interpreter)在本博客中，V8 支持的所有设备和平台。

这些改进只是一个开始。新的Ignition和TurboFan管道为进一步的优化铺平了道路，这些优化将提高JavaScript性能，并在未来几年内缩小V8在Chrome和Node.js中的占用空间。我们期待在向开发人员和用户推出这些改进时与您分享这些改进。敬请期待。
