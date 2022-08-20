***

标题： 'V8 版本 v6.2'
作者： 'V8团队'
日期： 2017-09-11 13：33：37
标签：

*   释放
    描述： “V8 v6.2 包括性能改进、更多 JavaScript 语言功能、增加的最大字符串长度等等。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.2](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.2)，直到几周后与Chrome 62 Stable合作发布之前，它一直处于测试阶段。V8 v6.2 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 性能改进

性能[`Object#toString`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/toString)以前已经被确定为潜在的瓶颈，因为它经常被流行的库使用，例如[洛达什](https://lodash.com/)和[下划线.js](http://underscorejs.org/)，以及类似[AngularJS](https://angularjs.org/).各种辅助函数，如[`_.isPlainObject`](https://github.com/lodash/lodash/blob/6cb3460fcefe66cb96e55b82c6febd2153c992cc/isPlainObject.js#L13-L50),[`_.isDate`](https://github.com/lodash/lodash/blob/6cb3460fcefe66cb96e55b82c6febd2153c992cc/isDate.js#L8-L25),[`angular.isArrayBuffer`](https://github.com/angular/angular.js/blob/464dde8bd12d9be8503678ac5752945661e006a5/src/Angular.js#L739-L741)或[`angular.isRegExp`](https://github.com/angular/angular.js/blob/464dde8bd12d9be8503678ac5752945661e006a5/src/Angular.js#L680-L689)通常在整个应用程序和库代码中使用，以执行运行时类型检查。

随着ES2015的到来，`Object#toString`通过新的[`Symbol.toStringTag`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/toStringTag)符号，这也使`Object#toString`重量更重，加速更具挑战性。在此版本中，我们移植了最初在[SpiderMonkey JavaScript 引擎](https://bugzilla.mozilla.org/show_bug.cgi?id=1369042#c0)到 V8，加快吞吐量`Object#toString`按系数**6，5×**.

![](/\_img/v8-release-62/perf.svg)

它还会影响Speedometer浏览器基准测试，特别是AngularJS子测试，我们测量了3%的稳定改进。阅读[详细的博客文章](https://ponyfoo.com/articles/investigating-performance-object-prototype-to-string-es2015)以获取更多信息。

![](/\_img/v8-release-62/speedometer.svg)

我们还显著提高了[ES2015 代理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)，通过以下方式加快调用代理对象的速度`someProxy(params)`或`new SomeOtherProxy(params)`最多**5×**:

![](/\_img/v8-release-62/proxy-call-construct.svg)

同样，通过以下方式访问代理对象上的属性的性能`someProxy.property`改进为几乎**6，5×**:

![](/\_img/v8-release-62/proxy-property.svg)

这是正在进行的实习的一部分。请继续关注更详细的博客文章和最终结果。

我们也很高兴地宣布，感谢[贡献](https://chromium-review.googlesource.com/c/v8/v8/+/620150)从[黄彼得](https://twitter.com/peterwmwong)、性能[`String#includes`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes)内置改进超过**3×**自上一版本以来。

内部哈希表的哈希码查找速度大大加快，从而提高了`Map`,`Set`,`WeakMap`和`WeakSet`.即将发表的博客文章将详细解释此优化。

![](/\_img/v8-release-62/hashcode-lookups.png)

垃圾回收器现在使用[平行清道夫](https://bugs.chromium.org/p/chromium/issues/detail?id=738865)用于收集堆的所谓年轻一代。

## 增强的低内存模式

在过去的几个版本中，V8的低内存模式得到了增强（例如[将初始半空间大小设置为 512 KB](https://chromium-review.googlesource.com/c/v8/v8/+/594387)).低内存设备现在遇到的内存不足情况更少。但是，这种内存不足的行为可能会对运行时性能产生负面影响。

## 更多正则表达式功能

支持[这`dotAll`模式](https://github.com/tc39/proposal-regexp-dotall-flag)对于正则表达式，通过`s`标记，现在默认启用。在`dotAll`模式，`.`正则表达式中的原子匹配任何字符，包括行终止符。

```js
/foo.bar/su.test('foo\nbar'); // true
```

[查看断言](https://github.com/tc39/proposal-regexp-lookbehind)，另一个新的正则表达式功能，现在默认可用。这个名字已经很好地描述了它的含义。Lookbehind 断言提供了一种方法，用于将模式限制为仅在 lookbehind 组中的模式前面才匹配。它有匹配和不匹配的风格：

```js
/(?<=\$)\d+/.exec('$1 is worth about ¥123'); // ['1']
/(?<!\$)\d+/.exec('$1 is worth about ¥123'); // ['123']
```

有关这些功能的更多详细信息，请参阅我们的博客文章，标题为[即将推出的正则表达式功能](https://developers.google.com/web/updates/2017/07/upcoming-regexp-features).

## 模板文本修订

模板文本中对转义序列的限制已放宽[根据相关提案](https://tc39.es/proposal-template-literal-revision/).这为模板标签提供了新的用例，例如编写LaTeX处理器。

```js
const latex = (strings) => {
  // …
};

const document = latex`
\newcommand{\fun}{\textbf{Fun!}}
\newcommand{\unicode}{\textbf{Unicode!}}
\newcommand{\xerxes}{\textbf{King!}}
Breve over the h goes \u{h}ere // Illegal token!
`;
```

## 增加的最大字符串长度

64 位平台上的最大字符串长度从`2**28 - 16`自`2**30 - 25`字符。

## 全编码机消失了

在 V8 v6.2 中，旧管道的最后主要部分消失了。此版本中删除了超过 30K 行代码 ， 这在降低代码复杂性方面明显获胜。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.2 -t branch-heads/6.2`以试验 V8 v6.2 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
