***

标题： “退休辛烷值”
作者： 'V8团队'
日期： 2017-04-12 13：33：37
标签：

*   基准
    描述： “V8 团队认为，是时候停用 Octane 作为推荐基准了。

***

JavaScript 基准测试的历史是一个不断发展的故事。随着 Web 从简单的文档扩展到动态客户端应用程序，新的 JavaScript 基准测试被创建来衡量对新用例变得重要的工作负载。这种不断的变化赋予了单个基准的有限寿命。随着Web浏览器和虚拟机（VM）实现开始针对特定测试用例进行过度优化，基准测试本身不再成为其原始用例的有效代理。最早的 JavaScript 基准测试之一，[太阳蜘蛛](https://webkit.org/perf/sunspider/sunspider.html)，为发布快速优化编译器提供了早期的激励。然而，随着VM工程师的发现[微基准标记的局限性](https://blog.mozilla.org/nnethercote/2014/06/16/a-browser-benchmarking-manifesto/)并找到了新的方法来[优化](https://benediktmeurer.de/2016/12/16/the-truth-about-traditional-javascript-benchmarks/#the-notorious-sunspider-examples) [周围](https://bugzilla.mozilla.org/show_bug.cgi?id=787601)太阳蜘蛛[局限性](https://bugs.webkit.org/show_bug.cgi?id=63864)、浏览器社区[退休](https://trac.webkit.org/changeset/187526/webkit)SunSpider 作为推荐的基准测试。

## 辛烷的起源

旨在减轻早期微弯标记的一些弱点，[辛烷基准测试套件](https://developers.google.com/octane/)于2012年首次发布。它从早期的一组简单演变而来[V8 测试用例](http://www.netchain.com/Tools/v8/)并成为一般Web性能的共同基准。Octane 由 17 个不同的测试组成，旨在涵盖各种不同的工作负载，从 Martin Richards 的内核模拟测试到[微软的TypeScript编译器](http://www.typescriptlang.org/)编译自身。Octane 的内容代表了在 JavaScript 创建时衡量 JavaScript 性能的主流智慧。

## 收益递减和过度优化

在发布后的头几年，Octane 为 JavaScript VM 生态系统提供了独特的价值。它允许包括V8在内的发动机针对一类强调峰值性能的应用优化其性能。这些 CPU 密集型工作负荷最初由 VM 实现提供不足服务。Octane 帮助引擎开发人员提供优化，使计算量大的应用程序能够达到使 JavaScript 成为C++或 Java 的可行替代方案的速度。此外，Octane 推动了垃圾回收方面的改进，帮助 Web 浏览器避免了长时间或不可预测的暂停。

然而，到2015年，大多数JavaScript实现已经实现了在Octane上获得高分所需的编译器优化。努力在Octane上获得更高的基准测试分数转化为真实网页性能的日益微弱的改进。对运行的执行配置文件的调查[辛烷值与加载常见网站](/blog/real-world-performance)（如Facebook，Twitter或维基百科）透露，基准测试不会执行V8的[解析 器](https://medium.com/dev-channel/javascript-start-up-performance-69200f43b201#.7v8b4jylg)或浏览器[装载堆栈](https://medium.com/reloading/toward-sustainable-loading-4760957ee46f#.muk9kzxmb)现实世界代码的方式。此外，Octane 的 JavaScript 风格与大多数现代框架和库采用的习语和模式不匹配（更不用说转译代码或更新的 ES2015+ 语言功能）。这意味着使用 Octane 测量 V8 性能并不能捕获现代 Web 的重要用例，例如快速加载框架、使用新的状态管理模式支持大型应用程序，或者确保 ES2015+ 的功能[与 ES5 等效产品一样快](/blog/high-performance-es2015).

此外，我们开始注意到，JavaScript优化可以提高辛烷值，这往往会对现实世界的场景产生不利影响。Octane 鼓励主动内联以最大限度地减少函数调用的开销，但针对 Octane 量身定制的内联策略导致编译成本增加和实际用例中内存使用率增加而导致的倒退。即使优化在现实世界中可能真正有用，例如[动态预增强](http://dl.acm.org/citation.cfm?id=2754181)，追求更高的辛烷值可能会导致开发过于特定的启发式方法，这些启发式方法几乎没有影响，甚至在更通用的情况下会降低性能。我们发现，辛烷值衍生的预增强启发式方法导致[现代框架，如Ember](https://bugs.chromium.org/p/v8/issues/detail?id=3665).这`instanceof`运算符是另一个针对一组有限的辛烷值特定情况量身定制的优化示例，这些情况导致[Node.js应用程序中的重大回归](https://github.com/nodejs/node/issues/9634).

另一个问题是，随着时间的推移，Octane 中的小错误会成为优化本身的目标。例如，在 Box2DWeb 基准测试中，利用[一个错误](http://crrev.com/1355113002)其中，使用`<`和`>=`操作员对辛烷值的性能提升了约 15%。不幸的是，这种优化在现实世界中没有效果，并使更一般的比较优化类型复杂化。Octane 有时甚至会对现实世界的优化产生负面影响：工程师在其他虚拟机上工作[已注意到](https://bugzilla.mozilla.org/show_bug.cgi?id=1162272)Octane似乎惩罚了懒惰解析，鉴于在野外经常发现的死代码数量，这种技术可以帮助大多数真实网站更快地加载。

## 超越辛烷值和其他合成基准

这些示例只是许多优化中的一部分，这些优化增加了 Octane 分数，从而损害了运行真实网站。不幸的是，类似的问题存在于其他静态或合成基准测试中，包括Kraken和JetStream。简而言之，这样的基准测试不足以衡量实际速度，并为VM工程师过度优化狭窄的用例和优化不足的通用案例创造了激励，从而减慢了JavaScript代码的速度。

鉴于大多数 JS VM 的得分处于稳定状态，并且针对特定 Octane 基准进行优化与为更广泛的实际代码实施加速之间日益增加的冲突，我们认为现在是时候停用 Octane 作为推荐基准了。

Octane 使 JS 生态系统能够在计算成本高昂的 JavaScript 方面取得巨大收益。然而，下一个前沿领域是提高性能[真正的网页](/blog/real-world-performance)，现代图书馆，[框架](http://stateofjs.com/2016/frontend/)， ES2015+[语言功能](/blog/high-performance-es2015)，新模式[状态管理](http://redux.js.org/),[不可变对象分配](https://facebook.github.io/immutable-js/)和[模块](https://webpack.github.io/) [捆绑](http://browserify.org/).由于 V8 在许多环境中运行，包括 Node.js 中的服务器端，因此我们还投入时间了解实际的 Node 应用程序，并通过工作负载（如[阿克美尔](https://github.com/acmeair/acmeair-nodejs).

请返回此处查看有关以下内容的更多帖子[改进我们的测量方法](/blog/real-world-performance)和[新工作负载](/blog/optimizing-v8-memory)更好地代表实际性能。我们很高兴继续追求对用户和开发人员最重要的性能！
