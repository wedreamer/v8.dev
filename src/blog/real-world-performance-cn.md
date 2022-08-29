***

标题： “V8 如何衡量实际性能”
作者： 'V8团队'
日期： 2016-12-21 13：33：37
标签：

*   基准
    描述： “V8 团队开发了一种新方法来衡量和理解现实世界的 JavaScript 性能。

***

在过去的一年里，V8团队开发了一种新的方法来衡量和理解现实世界的JavaScript性能。我们利用从中收集到的见解来改变 V8 团队如何使 JavaScript 更快。我们新的现实世界焦点代表了我们传统性能重点的重大转变。我们相信，随着我们在 2017 年继续应用这种方法，它将显著提高用户和开发人员在 Chrome 和 Node.js中依赖 V8 的可预测性能来实现实际 JavaScript 的能力。

古老的格言“被测量的东西得到改进”在JavaScript虚拟机（VM）开发领域尤其正确。选择正确的指标来指导性能优化是 VM 团队随着时间的推移可以执行的最重要的事情之一。下面的时间轴大致说明了自 V8 初始发布以来 JavaScript 基准测试是如何演变的：

![Evolution of JavaScript benchmarks](../_img/real-world-performance/evolution.png)

从历史上看，V8和其他JavaScript引擎使用综合基准测试来衡量性能。最初，VM开发人员使用微基准标记，例如[太阳蜘蛛](https://webkit.org/perf/sunspider/sunspider.html)和[海肯](http://krakenbenchmark.mozilla.org/).随着浏览器市场的成熟，第二个基准测试时代开始了，在此期间，他们使用了更大但仍然综合测试套件，例如[辛烷](http://chromium.github.io/octane/)和[捷讯科技](http://browserbench.org/JetStream/).

微基准和静态测试套件有一些好处：它们易于引导，易于理解，并且能够在任何浏览器中运行，从而使比较分析变得容易。但这种便利性带来了许多缺点。由于它们包含的测试用例数量有限，因此很难设计出准确反映整个 Web 特征的基准测试。此外，基准通常不经常更新。因此，他们往往很难跟上JavaScript开发的新趋势和模式。最后，多年来，VM 作者探索了传统基准测试的每个角落和裂缝，在此过程中，他们发现并利用了在基准测试执行期间随机切换甚至跳过外部不可观察的工作来提高基准测试分数的机会。这种基准测试分数驱动的改进和基准测试的过度优化并不总是能提供太多面向用户或开发人员的好处，历史表明，从长远来看，很难制作一个“不可游戏”的综合基准测试。

## 测量真实网站：WebPageReplay和运行时调用统计

鉴于我们只看到传统静态基准测试的性能故事的一部分，V8团队开始通过对实际网站的加载进行基准测试来衡量实际性能。我们希望衡量反映最终用户实际浏览 Web 方式的用例，因此我们决定从 Twitter、Facebook 和 Google 地图等网站获取性能指标。使用一个名为[网页回放](https://github.com/chromium/web-page-replay)我们能够确定性地记录和重播页面加载。

同时，我们开发了一个名为 Runtime Call Stats 的工具，它允许我们分析不同的 JavaScript 代码如何强调不同的 V8 组件。这是第一次，我们不仅能够轻松地针对真实网站测试 V8 更改，而且能够充分了解 V8 在不同工作负载下如何以及为什么以不同的方式执行。

现在，我们针对大约 25 个网站的测试套件监控更改，以指导 V8 优化。除了上述网站和Alexa Top 100中的其他网站外，我们还选择了使用通用框架（React，Polymer，Angular，Ember等）实现的网站，来自各种不同地理区域的网站，以及开发团队与我们合作的网站或库，例如Wikipedia，Reddit，Twitter和Webpack。我们相信这 25 个站点代表了整个 Web，这些站点的性能改进将直接反映在 JavaScript 开发人员今天编写的站点的类似加速中。

有关我们网站测试和运行时调用统计信息的开发的详细演示，请参阅[BlinkOn 6 关于实际性能的演示](https://www.youtube.com/watch?v=xCx4uC7mn6Y).您甚至可以[自行运行运行时调用统计信息工具](/docs/rcs).

## 做出真正的改变

通过分析这些新的、真实的性能指标，并使用运行时调用统计将其与传统基准进行比较，也让我们更深入地了解各种工作负载如何以不同的方式对 V8 施加压力。

从这些测量中，我们发现 Octane 性能实际上是我们 25 个测试网站中大多数网站性能的不良代表。您可以在下面的图表中看到：Octane 的色条分布与任何其他工作负载都非常不同，尤其是那些用于现实世界网站的工作负载。运行 Octane 时，V8 的瓶颈通常是 JavaScript 代码的执行。然而，大多数现实世界的网站反而强调V8的解析器和编译器。我们意识到，针对 Octane 进行的优化通常对现实世界的网页缺乏影响，在某些情况下，这些影响也会降低。[优化使现实世界的网站变慢](https://benediktmeurer.de/2016/12/16/the-truth-about-traditional-javascript-benchmarks/#a-closer-look-at-octane).

![Distribution of time running all of Octane, running the line-items of Speedometer, and loading websites from our test suite on Chrome 57](../_img/real-world-performance/startup-distribution.png)

我们还发现，另一个基准测试实际上是真实网站的更好代理。[速度计](http://browserbench.org/Speedometer/)，WebKit 基准测试包括用 React、Angular、Ember 和其他框架编写的应用程序，演示了与 25 个站点非常相似的运行时配置文件。虽然没有基准测试可以与真实网页的保真度相匹配，但我们相信Speedometer在近似Web上现代JavaScript的真实工作负载方面比Octane做得更好。

## 底线：为所有人提供更快的 V8

在过去的一年中，真实的网站测试套件和我们的运行时调用统计工具使我们能够提供V8性能优化，将页面加载速度平均提高10-20%。鉴于过去一直专注于优化Chrome的页面加载，2016年该指标的两位数改进是一项重大成就。同样的优化也使我们在车速表上的得分提高了20-30%。

这些性能改进应该反映在Web开发人员使用现代框架和类似的JavaScript模式编写的其他站点中。我们对内置功能的改进，例如`Object.create`和[`Function.prototype.bind`](https://benediktmeurer.de/2015/12/25/a-new-approach-to-function-prototype-bind/)，围绕对象工厂模式进行优化，在V8上工作[内联缓存](https://en.wikipedia.org/wiki/Inline_caching)，并且正在进行的解析器改进旨在对所有开发人员使用的JavaScript的未被低估区域进行普遍适用的改进，而不仅仅是我们跟踪的代表性站点。

我们计划扩展对真实网站的使用，以指导V8性能工作。请继续关注有关基准测试和脚本性能的更多见解。
