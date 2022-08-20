***

标题： 'V8 发布 v5.0'
作者： 'V8团队'
发布日期：2016-03-15 13：33：37
标签：

*   释放
    描述： “V8 v5.0 附带了性能改进，并添加了对几个新的 ES2015 语言功能的支持。

***

V8 的第一步[发布流程](/docs/release-process)是 Git 主节点的一个新分支，紧接在 Chrome Beta 分支之前的 Chrome Beta 里程碑（大约每六周一次）。我们最新的发布分支是[V8 v5.0](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.0)，它将一直处于测试阶段，直到我们与Chrome 50 Stable一起发布稳定版本。以下是此版本 V8 中面向开发人员的新功能的亮点。

：：：备注
**注意：**版本号 5.0 不具有语义意义，也不标记主要版本（与次要版本相反）。
:::

## 改进的 ECMAScript 2015 （ES6） 支持

V8 v5.0 包含许多与正则表达式 （regex） 匹配相关的 ES2015 功能。

### RegExp Unicode 标志

这[RegExp Unicode 标志](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp#Parameters),`u`，将打开新的 Unicode 模式以进行正则表达式匹配。Unicode 标志将模式和正则表达式字符串视为一系列 Unicode 代码点。它还公开了 Unicode 代码点转义的新语法。

```js
/😊{2}/.test('😊😊');
// false

/😊{2}/u.test('😊😊');
// true

/\u{76}\u{38}/u.test('v8');
// true

/\u{1F60A}/u.test('😊');
// true
```

这`u`标志也使`.`atom（也称为单字符匹配器）匹配任何 Unicode 符号，而不仅仅是基本多语言平面 （BMP） 中的字符。

```js
const string = 'the 🅛 train';

/the\s.\strain/.test(string);
// false

/the\s.\strain/u.test(string);
// true
```

### 正则表达式自定义挂钩

ES2015 包含 RegExp 子类的钩子，用于更改匹配的语义。子类可以重写名为`Symbol.match`,`Symbol.replace`,`Symbol.search`和`Symbol.split`为了改变 RegExp 子类相对于`String.prototype.match`和类似的方法。

## ES2015 和 ES5 功能的性能改进

5.0 版还为已实现的 ES2015 和 ES5 功能带来了一些显著的性能改进。

rest 参数的实现速度比上一版本快 8-10 倍，这使得在函数调用后将大量参数收集到单个数组中更加高效。`Object.keys`，可用于以返回的相同顺序循环访问对象的可枚举属性`for`-`in`，现在速度大约快了 2 倍。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 5.0 -t branch-heads/5.0`以试验 V8 5.0 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
