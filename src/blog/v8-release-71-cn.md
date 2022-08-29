***

标题： 'V8 版本 v7.1'
作者： 斯蒂芬·赫尔胡特 （[@herhut](https://twitter.com/herhut)），克隆克隆的克隆克隆者
化身：

*   斯蒂芬-赫胡特
    日期： 2018-10-31 15：44：37
    标签：
*   释放
    描述： 'V8 v7.1 具有嵌入式字节码处理程序，改进的 TurboFan 转义分析、postMessage（wasmModule）、Intl.RelativeTimeFormat 和 globalThis！'
    推文：“1057645773465235458”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.1](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.1)，直到几周后与Chrome 71 Stable一起发布之前，它一直处于测试阶段。V8 v7.1 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 记忆

在 v6.9/v7.0 中的工作之后[将内置内容直接嵌入到二进制文件中](/blog/embedded-builtins)，解释器的字节码处理程序现在也是[嵌入到二进制文件中](https://bugs.chromium.org/p/v8/issues/detail?id=8068).这平均为每个隔离株节省了约 200 KB。

## 性能

TurboFan 中的逃逸分析（用于对优化单元的本地对象执行标量替换）已改进为[处理高阶函数的局部函数上下文](https://bit.ly/v8-turbofan-context-sensitive-js-operators)当来自周围上下文的变量逃逸到局部闭包时。请考虑以下示例：

```js
function mapAdd(a, x) {
  return a.map(y => y + x);
}
```

请注意，`x`是局部闭包的自由变量`y => y + x`.V8 v7.1 现在可以完全消除`x`，最多可提高**40%**在某些情况下。

![Performance improvement with new escape analysis (lower is better)](../_img/v8-release-71/improved-escape-analysis.svg)

转义分析现在还能够消除变量索引访问局部数组的某些情况。下面是一个示例：

```js
function sum(...args) {
  let total = 0;
  for (let i = 0; i < args.length; ++i)
    total += args[i];
  return total;
}

function sum2(x, y) {
  return sum(x, y);
}
```

请注意，`args`是本地的`sum2`（假设`sum`内联到`sum2`).在 V8 v7.1 中，TurboFan 现在可以消除`args`完全替换变量索引访问`args[i]`具有以下形式的三元运算`i === 0 ? x : y`.这比JetStream/EarleyBoyer基准测试提高了约2%。将来，我们可能会将此优化扩展到具有两个以上元素的数组。

## Wasm 模块的结构化克隆

最后[`postMessage`支持 Wasm 模块](https://github.com/WebAssembly/design/pull/1074).`WebAssembly.Module`对象现在可以是`postMessage`'d 给网络工作者。为了澄清，这仅限于Web工作线程（相同的进程，不同的线程），而不是扩展到跨进程方案（例如跨源）`postMessage`或共享的 Web 工作者）。

## JavaScript 语言特性

[这`Intl.RelativeTimeFormat`应用程序接口](/features/intl-relativetimeformat)启用相对时间的本地化格式（例如“昨天”、“42 秒前”或“3 个月内”），而不会牺牲性能。下面是一个示例：

```js
// Create a relative time formatter for the English language that does
// not always have to use numeric value in the output.
const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

rtf.format(-1, 'day');
// → 'yesterday'

rtf.format(0, 'day');
// → 'today'

rtf.format(1, 'day');
// → 'tomorrow'

rtf.format(-1, 'week');
// → 'last week'

rtf.format(0, 'week');
// → 'this week'

rtf.format(1, 'week');
// → 'next week'
```

读[我们`Intl.RelativeTimeFormat`解释器](/features/intl-relativetimeformat)了解更多信息。

V8 v7.1 还添加了对[这`globalThis`建议](/features/globalthis)，使通用机制即使在严格的函数或模块中也可以访问全局对象，而不管平台如何。

## V8 接口

请使用`git log branch-heads/7.0..branch-heads/7.1 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.1 -t branch-heads/7.1`以试验 V8 v7.1 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
