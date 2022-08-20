***

标题：“Chrome 欢迎 Speedometer 2.0！
作者：《眨眼》和《V8战队》
日期： 2018-01-24 13：33：37
标签：

*   基准
    描述： “概述到目前为止我们在基于速度计 2.0 的 Blink 和 V8 中所做的性能改进。
    推文：“956232641736421377”

***

自 2014 年首次发布 Speedometer 1.0 以来，Blink 和 V8 团队一直在使用基准测试作为实际使用流行 JavaScript 框架的代理，我们在此基准测试上实现了可观的加速。我们独立验证了这些改进通过衡量现实世界的网站来转化为真实的用户利益，并观察到热门网站页面加载时间的改善也提高了速度计分数。

与此同时，JavaScript迅速发展，在ES2015和更高版本的标准中增加了许多新的语言功能。框架本身也是如此，因此速度计1.0随着时间的推移已经过时了。因此，使用速度计 1.0 作为优化指标会增加无法测量正在使用的较新代码模式的风险。

Blink和V8团队欢迎[最近发布的更新的车速表 2.0 基准测试](https://webkit.org/blog/8063/speedometer-2-0-a-benchmark-for-modern-web-app-responsiveness/).将原始概念应用于当代框架，转译器和ES2015功能列表，使基准测试再次成为优化的主要候选者。车速表 2.0 是一个很好的补充[我们实际性能基准工具带](/blog/real-world-performance).

## 到目前为止，Chrome的里程数

Blink和V8团队已经完成了第一轮改进，这证明了这个基准测试对我们的重要性，并继续我们专注于现实世界性能的旅程。将2017年7月的Chrome 60与最新的Chrome 64进行比较，我们在2016年中期的Macbook Pro（4核，16GB RAM）上的总得分（每分钟运行次数）提高了约21%。

![Comparison of Speedometer 2 scores between Chrome 60 and 64](/\_img/speedometer-2/scores.png)

让我们放大各个车速表 2.0 行项目。我们通过改进将 React 运行时的性能提高了一倍[`Function.prototype.bind`](https://chromium.googlesource.com/v8/v8/+/808dc8cff3f6530a627ade106cbd814d16a10a18).Vanilla-ES2015、AngularJS、Preact 和 VueJS 改进了 19%–42%，原因是[加快 JSON 解析速度](https://chromium-review.googlesource.com/c/v8/v8/+/700494)以及各种其他性能修复。jQuery-TodoMVC应用程序的运行时通过改进Blink的DOM实现而减少，包括[更轻量级的表单控件](https://chromium.googlesource.com/chromium/src/+/f610be969095d0af8569924e7d7780b5a6a890cd)和[调整到我们的 HTML 解析器](https://chromium.googlesource.com/chromium/src/+/6dd09a38aaae9c15adf5aad966f761f180bf1cef).V8 的内联缓存的额外调整与优化编译器相结合，带来了全面的改进。

![Score improvements for each Speedometer 2 subtest from Chrome 60 to 64](/\_img/speedometer-2/improvements.png)

与车速表1.0相比，一个显着的变化是最终分数的计算。以前，所有分数的平均值都倾向于只处理最慢的行项目。例如，在查看每个行项目上花费的绝对时间时，我们看到EmberJS-Debug版本花费的时间大约是最快的基准测试的35倍。因此，提高总体得分，专注于EmberJS-Debug具有最大的潜力。

![](/\_img/speedometer-2/time.png)

车速表2.0使用几何平均值作为最终得分，有利于对每个框架进行平等投资。让我们考虑一下我们最近从上面改进了16.5%的Preact。仅仅因为它对总时间的微小贡献而放弃16.5%的改进是相当不公平的。

我们期待为 Speedometer 2.0 以及通过它为整个网络带来进一步的性能改进。请继续关注更多性能击掌。
