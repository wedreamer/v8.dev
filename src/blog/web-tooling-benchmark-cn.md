***

标题： “宣布 Web 工具基准测试”
作者： 'Benedikt Meurer （[@bmeurer](https://twitter.com/bmeurer)），JavaScript Performance Juggler'
化身：

*   'benedikt-meurer'
    日期： 2017-11-06 13：33：37
    标签：
*   基准
*   节点.js
    描述： “全新的 Web 工具基准测试有助于识别和修复 Babel、TypeScript 和其他实际项目中的 V8 性能瓶颈。
    推文：“927572065598824448”

***

JavaScript 性能对 V8 团队一直很重要，在这篇文章中，我们想讨论一个新的 JavaScript。[网络工具基准测试](https://v8.github.io/web-tooling-benchmark)我们最近一直在使用它来识别和修复 V8 中的一些性能瓶颈。您可能已经知道 V8 的[对 Node 的坚定承诺.js](/blog/v8-nodejs)此基准测试通过专门运行基于基于 Node.js 构建的常见开发人员工具的性能测试来扩展这一承诺。Web 工具基准测试中的工具与当今开发人员和设计人员用于构建现代网站和基于云的应用程序的工具相同。继续我们不断努力，专注于[实际性能](/blog/real-world-performance/)我们不是人为的基准测试，而是使用开发人员每天运行的实际代码创建了基准测试。

Web 工具基准测试套件从一开始就设计为涵盖重要的[开发人员工具用例](https://github.com/nodejs/benchmarking/blob/master/docs/use_cases.md#web-developer-tooling)对于节点.js。由于 V8 团队专注于核心 JavaScript 性能，因此我们构建基准测试的方式侧重于 JavaScript 工作负载，并且排除了对 Node.js特定 I/O 或外部交互的度量。这使得可以在Node.js，所有浏览器和所有主要JavaScript引擎shell中运行基准测试，包括`ch`（脉轮核心），`d8`（V8），`jsc`（JavaScriptCore） 和`jsshell`（蜘蛛猴）。尽管基准测试不仅限于Node.js，但我们很高兴[节点.js基准测试工作组](https://github.com/nodejs/benchmarking)正在考虑使用工具基准测试作为 Node 性能的标准（[nodejs/benchmarking#138](https://github.com/nodejs/benchmarking/issues/138)).

工具基准测试中的单个测试涵盖了开发人员通常用于构建基于 JavaScript 的应用程序的各种工具，例如：

*   这[巴别塔](https://github.com/babel/babel)转译器使用`es2015`预设。
*   Babel 使用的解析器 — 命名为[巴比伦](https://github.com/babel/babylon)— 运行在几个流行的输入（包括[洛达什](https://lodash.com/)和[预作用](https://github.com/developit/preact)捆绑包）。
*   这[橡子](https://github.com/ternjs/acorn)使用的解析器[网络包](http://webpack.js.org/).
*   这[TypeScript](http://www.typescriptlang.org/)编译器在 上运行[打字-角度](https://github.com/tastejs/todomvc/tree/master/examples/typescript-angular)示例项目来自[TodoMVC](https://github.com/tastejs/todomvc)项目。

查看[深入分析](https://github.com/v8/web-tooling-benchmark/blob/master/docs/in-depth.md)有关所有包含的测试的详细信息。

基于过去使用其他基准测试的经验，例如[速度计](http://browserbench.org/Speedometer)，随着新版本的框架可用，测试很快就会过时，我们确保在发布基准测试中的每个工具时将它们更新到更新到更新的版本是直截了当的。通过将基准测试套件基于npm基础架构，我们可以轻松更新它，以确保它始终在JavaScript开发工具中测试最先进的技术。更新测试用例只是将版本放在`package.json`清单。

我们创建了一个[跟踪错误](http://crbug.com/v8/6936)和[电子表格](https://docs.google.com/spreadsheets/d/14XseWDyiJyxY8\_wXkQpc7QCKRgMrUbD65sMaNvAdwXw)包含我们收集的有关 V8 在新基准测试中性能的所有相关信息。我们的调查已经产生了一些有趣的结果。例如，我们发现 V8 经常在`instanceof`([v8：6971](http://crbug.com/v8/6971)），导致 3–4×减速。我们还发现并修复了某些情况下的性能瓶颈`obj[name] = val`哪里`obj`通过以下方式创建：`Object.create(null)`.在这些情况下，V8 会从快速路径上掉下来，尽管它能够利用以下事实：`obj`具有`null`原型（[v8：6985](http://crbug.com/v8/6985)).在这个基准测试的帮助下，这些和其他发现不仅在Node.js中，而且在Chrome中都改进了V8。

我们不仅研究了如何使 V8 更快，而且还修复了基准测试的工具和库中的性能错误，并在发现它们时对其进行了上游处理。例如，我们在[巴别塔](https://github.com/babel/babel)其中代码模式如

```js
value = items[items.length - 1];
```

导致进入酒店`"-1"`，因为代码没有检查是否`items`事先为空。此代码模式导致 V8 通过慢速路径，因为`"-1"`查找，即使稍微修改，等效版本的JavaScript要快得多。我们帮助解决了巴别塔（[巴别/巴别#6582](https://github.com/babel/babel/pull/6582),[巴别/巴别#6581](https://github.com/babel/babel/pull/6581)和[巴别/巴别#6580](https://github.com/babel/babel/pull/6580)).我们还发现并修复了一个错误，其中Babel可以访问超过字符串长度（[巴别/巴别#6589](https://github.com/babel/babel/pull/6589)），这在 V8 中触发了另一条慢速路径。此外，我们[优化了数组和字符串的越界读取](https://twitter.com/bmeurer/status/926357262318305280)在 V8 中。我们期待继续[与社区合作](https://twitter.com/rauchg/status/924349334346276864)关于提高这个重要用例的性能，不仅在V8上运行时，而且在ChakraCore等其他JavaScript引擎上运行时。

我们非常关注实际性能，尤其是改进流行的 Node.js工作负载，这体现在 V8 在基准测试上的得分在过去几个版本中的不断提高：

![](../_img/web-tooling-benchmark/chart.svg)

自 V8 v5.8 起，这是之前的最后一个 V8 版本[切换到点火+涡轮风扇架构](/blog/launching-ignition-and-turbofan)，V8在工具基准测试上的得分提高了大约**60%**.

在过去的几年里，V8 团队已经认识到，没有一个 JavaScript 基准测试 —— 即使是一个善意的、精心设计的基准测试—— 应该被用作 JavaScript 引擎整体性能的单一代理。但是，我们确实认为**网络工具基准测试**重点介绍了值得关注的 JavaScript 性能领域。尽管有这个名字和最初的动机，我们发现Web工具基准套件不仅代表了工具工作负载，而且代表了大量更复杂的JavaScript应用程序，这些应用程序没有经过前端基准测试，如Speedometer。它绝不是车速表的替代品，而是一套补充测试。

最好的消息是，鉴于Web工具基准测试是如何围绕实际工作负载构建的，我们预计我们最近对基准测试分数的改进将直接转化为开发人员的生产力的提高，[减少等待构建的时间](https://xkcd.com/303/).Node.js中已经提供了许多这些改进：在撰写本文时，节点 8 LTS 处于 V8 v6.1，节点 9 处于 V8 v6.2。

最新版本的基准测试托管在<https://v8.github.io/web-tooling-benchmark/>.
