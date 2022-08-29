***

标题：“帮助我们测试 V8 的未来！
作者： '丹尼尔·克利福德 （[@expatdanno](https://twitter.com/expatdanno)），原装慕尼黑V8酿酒师
日期： 2017-02-14 13：33：37
标签：

*   内部
    描述： '今天在Chrome Canary中预览V8的新编译器管道，其中包含Ignition和TurboFan！

***

V8 团队目前正在开发一个新的默认编译器管道，它将帮助我们将未来的加速技术提升到[现实世界的 JavaScript](/blog/real-world-performance).您可以立即在 Chrome Canary 中预览新的管道，以帮助我们验证当我们为所有 Chrome 频道推出新配置时是否没有任何意外。

新的编译器管道使用[点火解释器](/blog/ignition-interpreter)和[涡轮风扇编译器](/docs/turbofan)执行所有JavaScript（代替由Full-codegen和Crankshaft编译器组成的经典管道）。Chrome Canary 和 Chrome Developer 频道用户的随机子集已经在测试新配置。但是，任何人都可以通过在 about：flags 中翻转标志来选择加入新管道（或恢复到旧管道）。

您可以通过选择加入新管道并在您喜欢的网站上将其与 Chrome 配合使用来帮助测试新管道。如果您是 Web 开发人员，请使用新的编译器管道测试 Web 应用程序。如果您发现稳定性，正确性或性能下降，请[将问题报告给 V8 错误跟踪器](https://bugs.chromium.org/p/v8/issues/entry?template=Bug%20report%20for%20the%20new%20pipeline).

## 如何启用新管道

### 在 Chrome 58 中

1.  安装最新的[试用版](https://www.google.com/chrome/browser/beta.html)
2.  打开网址`about:flags`在铬
3.  搜索”**实验性 JavaScript 编译管道**“并将其设置为”**启用**"

![](../_img/test-the-future/58.png)

### 在 Chrome 59.0.3056 及更高版本中

1.  安装最新的金丝雀[金丝雀](https://www.google.com/chrome/browser/canary.html)或[开发](https://www.google.com/chrome/browser/desktop/index.html?extra=devchannel)
2.  打开网址`about:flags`在铬
3.  搜索”**经典 JavaScript 编译管道**“并将其设置为”**禁用**"

![](../_img/test-the-future/59.png)

标准值为”**违约**“，这意味着**或**经典管道处于活动状态，具体取决于 A/B 测试配置。

## 如何报告问题

如果在默认管道上使用新管道时，您的浏览体验是否有显著变化，请告知我们。如果您是 Web 开发人员，请在（移动）Web 应用程序上测试新管道的性能，以了解它是如何受到影响的。如果您发现您的 Web 应用程序行为异常（或测试失败），请告诉我们：

1.  确保已正确启用新管道，如上一节中所述。
2.  [在 V8 的错误跟踪器上创建错误](https://bugs.chromium.org/p/v8/issues/entry?template=Bug%20report%20for%20the%20new%20pipeline).
3.  附加示例代码，我们可以使用它来重现问题。
