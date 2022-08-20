***

标题： 'V8 版本 v6.4'
作者： 'V8团队'
日期： 2017-12-19 13：33：37
标签：

*   释放
    描述： “V8 v6.4 包括性能改进、新的 JavaScript 语言特性等。
    推文：“943057597481082880”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.4)，直到几周后与Chrome 64 Stable合作发布之前，它一直处于测试阶段。V8 v6.4 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 速度

V8 v6.4[提高](https://bugs.chromium.org/p/v8/issues/detail?id=6971)性能`instanceof`运算符 3.6×。直接结果是，[uglify-js](http://lisperator.net/uglifyjs/)现在速度提高了 15–20%，根据[V8 的 Web 工具基准测试](https://github.com/v8/web-tooling-benchmark).

此版本还解决了`Function.prototype.bind`.例如，TurboFan now[始终如一的内联](https://bugs.chromium.org/p/v8/issues/detail?id=6946)所有单态调用`bind`.此外，TurboFan还支持*绑定回调模式*，表示而不是以下内容：

```js
doSomething(callback, someObj);
```

您现在可以使用：

```js
doSomething(callback.bind(someObj));
```

这样，代码更具可读性，并且您仍然可以获得相同的性能。

由于[黄彼得](https://twitter.com/peterwmwong)的最新贡献，[`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)和[`WeakSet`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet)现在使用[CodeStubAssembler](/blog/csa)，从而全面提高高达 5× 的性能。

![](/\_img/v8-release-64/weak-collection.svg)

作为 V8 的一部分[持续的努力](https://bugs.chromium.org/p/v8/issues/detail?id=1956)为了提高阵列内置的性能，我们改进了`Array.prototype.slice`性能 ~4×通过使用 CodeStubAssembler 重新实现它。此外，调用`Array.prototype.map`和`Array.prototype.filter`现在，许多案例都内联了内联，使它们具有与手写版本相媲美的性能配置文件。

我们努力在数组、类型化数组和字符串中进行越界加载[不再产生 ~10× 的性能影响](https://bugs.chromium.org/p/v8/issues/detail?id=7027)注意到后[此编码模式](/blog/elements-kinds#avoid-reading-beyond-length)在野外使用。

## 记忆

V8 的内置代码对象和字节码处理程序现在从快照中延迟反序列化，这可以显著减少每个隔离项消耗的内存。Chrome 中的基准测试显示，在浏览常见网站时，每个标签页可节省数百 KB。

![](/\_img/v8-release-64/codespace-consumption.svg)

请留意明年初关于这个主题的专门博客文章。

## ECMAScript 语言特性

此 V8 版本包括对两个新的令人兴奋的正则表达式功能的支持。

在正则表达式中，具有`/u`旗[Unicode 属性转义](https://mathiasbynens.be/notes/es-unicode-property-escapes)现在默认启用。

```js
const regexGreekSymbol = /\p{Script_Extensions=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

支持[命名捕获组](https://developers.google.com/web/updates/2017/07/upcoming-regexp-features#named_captures)中正则表达式现在默认处于启用状态。

```js
const pattern = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/u;
const result = pattern.exec('2017-12-15');
// result.groups.year === '2017'
// result.groups.month === '12'
// result.groups.day === '15'
```

有关这些功能的更多详细信息，请参阅我们的博客文章，标题为[即将推出的正则表达式功能](https://developers.google.com/web/updates/2017/07/upcoming-regexp-features).

由于[Groupon](https://twitter.com/GrouponEng)，V8 现在实现[`import.meta`](https://github.com/tc39/proposal-import-meta)，这使嵌入器能够公开有关当前模块的特定于主机的元数据。例如，Chrome 64 通过以下方式公开模块网址`import.meta.url`，以及 Chrome 计划向其添加更多属性`import.meta`在未来。

为了帮助对国际化格式化程序生成的字符串进行本地感知格式设置，开发人员现在可以使用[`Intl.NumberFormat.prototype.formatToParts()`](https://github.com/tc39/proposal-intl-formatToParts)将数字的格式设置为标记及其类型的列表。由于[伊加利亚](https://twitter.com/igalia)用于在 V8 中实现此内容！

## V8 接口

请使用`git log branch-heads/6.3..branch-heads/6.4 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.4 -t branch-heads/6.4`以试验 V8 v6.4 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
