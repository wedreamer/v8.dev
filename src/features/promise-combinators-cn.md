***

标题： “承诺组合器”
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-06-12
    标签：
*   ECMAScript
*   ES2020
*   ES2021
*   io19
*   节点.js 16
    描述：“JavaScript 中有四个 promise 组合器：Promise.all、Promise.race、Promise.allSettled 和 Promise.any。
    推文：“1138819493956710400”

***

自从在ES2015中引入承诺以来，JavaScript已经支持了两个承诺组合器：静态方法`Promise.all`和`Promise.race`.

目前，有两项新提案正在通过标准化进程：`Promise.allSettled`和`Promise.any`.通过这些添加，JavaScript中总共将有四个承诺组合器，每个组合器都支持不同的用例。

以下是四个组合器的概述：

：：：表包装器
|名称 |描述 |状态|
|------------------------------------------- |----------------------------------------------- |--------------------------------------------------------------- |
|[`Promise.allSettled`](#promise.allsettled)|不会短路|[ES2020 ✅ 中新增](https://github.com/tc39/proposal-promise-allSettled)|
|[`Promise.all`](#promise.all)|输入值被拒绝时的短路|ES2015 ✅ |中新增
|[`Promise.race`](#promise.race)|输入值建立时的短路|ES2015 ✅ |中新增
|[`Promise.any`](#promise.any)|满足输入值时的短路|[ES2021 ✅ 中新增](https://github.com/tc39/proposal-promise-any)|
:::

让我们看一下每个组合器的示例用例。

## `Promise.all`

<feature-support chrome="32"
              firefox="29"
              safari="8"
              nodejs="0.12"
              babel="yes https://github.com/zloirock/core-js#ecmascript-promise"></feature-support>

`Promise.all`让您知道何时所有输入承诺都已实现或当其中一个承诺被拒绝时。

想象一下，用户单击一个按钮，而您希望加载一些样式表，以便您可以呈现一个全新的 UI。此程序并行启动每个样式表的 HTTP 请求：

```js
const promises = [
  fetch('/component-a.css'),
  fetch('/component-b.css'),
  fetch('/component-c.css'),
];
try {
  const styleResponses = await Promise.all(promises);
  enableStyles(styleResponses);
  renderNewUi();
} catch (reason) {
  displayError(reason);
}
```

您只想开始渲染新 UI 一次*都*请求成功。如果出现问题，您希望尽快显示错误消息，而不等待其他任何其他工作完成。

在这种情况下，您可以使用`Promise.all`：你想知道所有的承诺何时实现，*或*只要其中一人拒绝。

## `Promise.race`

<feature-support chrome="32"
              firefox="29"
              safari="8"
              nodejs="0.12"
              babel="yes https://github.com/zloirock/core-js#ecmascript-promise"></feature-support>

`Promise.race`如果要运行多个承诺，则很有用，并且...

1.  用第一个成功的结果做一些事情（如果其中一个承诺兑现），*或*
2.  一旦其中一个承诺被拒绝，就做点什么。

也就是说，如果其中一个承诺被拒绝，您希望保留该拒绝以单独处理错误情况。下面的示例正是这样做的：

```js
try {
  const result = await Promise.race([
    performHeavyComputation(),
    rejectAfterTimeout(2000),
  ]);
  renderResult(result);
} catch (error) {
  renderError(error);
}
```

我们启动了一项计算成本高昂的任务，可能需要很长时间，但我们将其与2秒后拒绝的承诺进行竞争。根据要实现或拒绝的第一个承诺，我们在两个单独的代码路径中呈现计算结果或错误消息。

## `Promise.allSettled`

<feature-support chrome="76"
              firefox="71 https://bugzilla.mozilla.org/show_bug.cgi?id=1549176"
              safari="13"
              nodejs="12.9.0 https://nodejs.org/en/blog/release/v12.9.0/"
              babel="yes https://github.com/zloirock/core-js#ecmascript-promise"></feature-support>

`Promise.allSettled`当所有输入承诺都为*安定*，这意味着它们要么*实现*或*拒绝*.这在你不关心承诺的状态的情况下很有用，你只想知道工作何时完成，无论它是否成功。

例如，您可以启动一系列独立的 API 调用并使用`Promise.allSettled`要确保在执行其他操作（例如删除加载微调器）之前全部完成，请执行以下操作：

```js
const promises = [
  fetch('/api-call-1'),
  fetch('/api-call-2'),
  fetch('/api-call-3'),
];
// Imagine some of these requests fail, and some succeed.

await Promise.allSettled(promises);
// All API calls have finished (either failed or succeeded).
removeLoadingIndicator();
```

## `Promise.any`

<feature-support chrome="85 https://bugs.chromium.org/p/v8/issues/detail?id=9808"
              firefox="79 https://bugzilla.mozilla.org/show_bug.cgi?id=1568903"
              safari="14 https://bugs.webkit.org/show_bug.cgi?id=202566"
              nodejs="16"
              babel="yes https://github.com/zloirock/core-js#ecmascript-promise"></feature-support>

`Promise.any`一旦其中一个承诺实现，就会给你一个信号。这类似于`Promise.race`除了`any`当其中一个承诺被拒绝时，不会提前拒绝。

```js
const promises = [
  fetch('/endpoint-a').then(() => 'a'),
  fetch('/endpoint-b').then(() => 'b'),
  fetch('/endpoint-c').then(() => 'c'),
];
try {
  const first = await Promise.any(promises);
  // Any of the promises was fulfilled.
  console.log(first);
  // → e.g. 'b'
} catch (error) {
  // All of the promises were rejected.
  console.assert(error instanceof AggregateError);
  // Log the rejection values:
  console.log(error.errors);
  // → [
  //     <TypeError: Failed to fetch /endpoint-a>,
  //     <TypeError: Failed to fetch /endpoint-b>,
  //     <TypeError: Failed to fetch /endpoint-c>
  //   ]
}
```

此代码示例检查哪个终结点响应最快，然后记录它。仅当*都*的请求失败，我们最终在`catch`块，然后我们可以处理错误。

`Promise.any`拒绝可以同时表示多个错误。为了在语言级别支持此功能，一个新的错误类型称为`AggregateError`介绍。除了在上面的示例中的基本用法之外，`AggregateError`对象也可以以编程方式构造，就像其他错误类型一样：

```js
const aggregateError = new AggregateError([errorA, errorB, errorC], 'Stuff went wrong!');
```
