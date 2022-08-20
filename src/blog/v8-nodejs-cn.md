***

标题： 'V8 ❤️ Node.js'
作者：'Franziska Hinkelmann，Node Monkey Patcher'
发布日期：2016-12-15 13：33：37
标签：

*   节点.js
    描述： “这篇博客文章重点介绍了最近为使 Node.js在 V8 和 Chrome DevTools 中得到更好的支持所做的一些努力。

***

在过去的几年里，Node.js的受欢迎程度一直在稳步增长，我们一直在努力使Node.js变得更好。这篇博客文章重点介绍了最近在 V8 和 DevTools 中所做的一些工作。

## 调试节点.js在 DevTools 中

您现在可以[使用 Chrome 开发人员工具调试 Node 应用程序](https://medium.com/@paul_irish/debugging-node-js-nightlies-with-chrome-devtools-7c4a1b95ae27#.knjnbsp6t).Chrome DevTools团队将实现调试协议的源代码从Chromium迁移到V8，从而使Node Core更容易与调试器源代码和依赖项保持同步。其他浏览器供应商和IDE也使用Chrome调试协议，共同改善了使用Node时的开发人员体验。

## ES2015 加速

我们正在努力使V8比以往更快。[我们最近的许多性能工作都围绕着ES6功能展开。](/blog/v8-release-56)，包括 promise、生成器、析构函数和 rest/spread 运算符。由于 Node 6.2 及更高版本中的 V8 版本完全支持 ES6，因此 Node 开发人员可以“原生”使用新的语言功能，而无需 polyfill。这意味着 Node 开发人员通常是第一个从 ES6 性能改进中受益的人。同样，他们通常是第一个识别性能倒退的人。多亏了一个细心的Node社区，我们发现并修复了许多回归，包括性能问题[`instanceof`](https://github.com/nodejs/node/issues/9634),[`buffer.length`](https://github.com/nodejs/node/issues/9006),[长参数列表](https://github.com/nodejs/node/pull/9643)和[`let`/`const`](https://github.com/nodejs/node/issues/9729).

## 修复了节点.js`vm`模块和 REPL 即将推出

这[`vm`模块](https://nodejs.org/dist/latest-v7.x/docs/api/vm.html)已经有[一些长期存在的限制](https://github.com/nodejs/node/issues/6283).为了正确解决这些问题，我们扩展了 V8 API 以实现更直观的行为。我们很高兴地宣布，vm 模块改进是我们作为导师支持的项目之一[节点基金会的外展](https://nodejs.org/en/foundation/outreachy/).我们希望在不久的将来看到这个项目和其他项目取得更多进展。

## `async`/`await`

使用异步函数，您可以通过按顺序等待承诺来重写程序流，从而大大简化异步代码。`async`/`await`将降落在节点中[随着下一个 V8 更新](https://github.com/nodejs/node/pull/9618).我们最近在改进 promise 和生成器性能方面的工作有助于快速实现异步函数。与此相关的是，我们还致力于提供[承诺钩](https://bugs.chromium.org/p/v8/issues/detail?id=4643)，一组自省 API 需要[Node Async Hook API](https://github.com/nodejs/node-eps/pull/18).

## 想要尝试前沿的 Node.js？

如果您有兴趣在 Node 中测试最新的 V8 功能，并且不介意使用前沿、不稳定的软件，您可以尝试我们的集成分支[这里](https://github.com/v8/node/tree/vee-eight-lkgr).[V8 持续集成到节点中](https://ci.chromium.org/p/v8/builders/luci.v8.ci/V8%20Linux64%20-%20node.js%20integration)在 V8 命中 Node 之前.js，因此我们可以及早发现问题。但请注意，这比Node.js树的尖端更具实验性。
