***

标题： 'V8 版本 v5.6'
作者： 'V8团队'
日期： 2016-12-02 13：33：37
标签：

*   释放
    描述： “V8 v5.6 附带了一个新的编译器管道，性能改进，并增加了对 ECMAScript 语言功能的支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.6)，它将处于测试阶段，直到几周后与Chrome 56 Stable协同发布。V8 5.6 充满了各种面向开发人员的好东西，所以我们想在发布之前预览一些亮点。

## 用于ES.next（以及更多）的点火和涡轮风扇管道已发货

从 5.6 开始，V8 可以优化整个 JavaScript 语言。此外，许多语言特性都是通过 V8 中新的优化管道发送的。此管道使用 V8 的[点火解释器](/blog/ignition-interpreter)作为基准，并使用更强大的 V8 优化频繁执行的方法[涡轮风扇优化编译器](/docs/turbofan).新的管道激活新的语言功能（例如，ES2015和ES2016规范中的许多新功能）或当曲轴（[V8 的“经典”优化编译器](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html)） 无法优化方法（例如 try-catch，with）。

为什么我们只通过新的管道路由一些JavaScript语言功能？新的管道更适合优化JS语言的整个频谱（过去和现在）。它是一个更健康，更现代的代码库，它是专门为现实世界的用例设计的，包括在低内存设备上运行V8。

我们已经开始将Ignition/TurboFan与我们添加到V8中的最新ES.next功能（ES.next = ES2015及更高版本中指定的JavaScript功能）一起使用，并将在继续提高其性能时通过它路由更多功能。在中期，V8 团队的目标是将 V8 中的所有 JavaScript 执行切换到新的管道。但是，只要仍然存在 Crankshaft 运行 JavaScript 速度比新的 Ignition/TurboFan 流水线更快的实际用例，在短期内，我们将同时支持这两个流水线，以确保在 V8 中运行的 JavaScript 代码在所有情况下都尽可能快。

那么，为什么新的管道同时使用新的Ignition解释器和新的TurboFan优化编译器呢？快速高效地运行 JavaScript 需要在 JavaScript 虚拟机中具有多个机制或层，以执行低级的繁忙工作。例如，拥有一个快速开始执行代码的第一层，然后是花费更长时间编译热函数的第二个优化层，以便最大限度地提高运行时间较长的代码的性能，这很有用。

Ignition和TurboFan是V8的两个新的执行层，当一起使用时最有效。由于效率，简单性和大小方面的考虑，TurboFan旨在优化JavaScript方法，从[字节码](https://en.wikipedia.org/wiki/Bytecode)由V8的点火解释器制作。通过将两个组件设计为紧密协作，由于另一个组件的存在，可以对两者进行优化。因此，从5.6开始，TurboFan将优化的所有功能首先通过Ignition解释器运行。使用这种统一的Ignition/TurboFan管道可以优化过去无法优化的功能，因为它们现在可以利用TurboFan的优化过程。例如，通过路由[发电机](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function\*)通过Ignition和TurboFan，发电机的运行性能几乎增加了两倍。

有关V8采用Ignition和TurboFan的旅程的更多信息，请查看[Benedikt的專門博客文章](https://benediktmeurer.de/2016/11/25/v8-behind-the-scenes-november-edition/).

## 性能改进

V8 v5.6 在内存和性能占用方面提供了许多关键改进。

### 记忆引起的卡顿

[并发记住集筛选](https://bugs.chromium.org/p/chromium/issues/detail?id=648568)已推出：迈向一步[奥里诺科河](/blog/orinoco).

### 大大提高了 ES2015 的性能

开发人员通常在转译器的帮助下开始使用新的语言功能，因为存在两个挑战：向后兼容性和性能问题。

V8 的目标是缩小转译器与 V8 的“原生”ES.next 性能之间的性能差距，以消除后一个挑战。我们在使新语言功能的性能与其转译的ES5等效项相媲美方面取得了巨大进展。在此版本中，您会发现 ES2015 功能的性能明显快于以前的 V8 版本，并且在某些情况下，ES2015 功能的性能接近转译的 ES5 等效产品。

特别是[传播](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Operators/Spread_operator)运算符现在应该已准备好在本机使用。而不是写...

```js
// Like Math.max, but returns 0 instead of -∞ for no arguments.
function specialMax(...args) {
  if (args.length === 0) return 0;
  return Math.max.apply(Math, args);
}
```

...你现在可以写...

```js
function specialMax(...args) {
  if (args.length === 0) return 0;
  return Math.max(...args);
}
```

...并获得类似的性能结果。特别是 V8 v5.6 包括以下微基准测试的加速：

*   [解构](https://github.com/fhinkel/six-speed/tree/master/tests/destructuring)
*   [解构数组](https://github.com/fhinkel/six-speed/tree/master/tests/destructuring-array)
*   [destructuring-string](https://github.com/fhinkel/six-speed/tree/master/tests/destructuring-string)
*   [对于阵列](https://github.com/fhinkel/six-speed/tree/master/tests/for-of-array)
*   [发电机](https://github.com/fhinkel/six-speed/tree/master/tests/generator)
*   [传播](https://github.com/fhinkel/six-speed/tree/master/tests/spread)
*   [传播生成器](https://github.com/fhinkel/six-speed/tree/master/tests/spread-generator)
*   [展开-文字](https://github.com/fhinkel/six-speed/tree/master/tests/spread-literal)

有关 V8 v5.4 和 v5.6 之间的比较，请参阅下表。

![Comparing the ES2015 feature performance of V8 v5.4 and v5.6 with SixSpeed](../_img/v8-release-56/perf.png)

这仅仅是个开始。在即将发布的版本中还有很多内容要跟进！

## 语言功能

### `String.prototype.padStart`/`String.prototype.padEnd`

[`String.prototype.padStart`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/padStart)和[`String.prototype.padEnd`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/padEnd)是 ECMAScript 的最新阶段 4 添加内容。这些库函数在 v5.6 中正式发布。

：：：备注
**注意：**再次卸货。
:::

## 网页组件浏览器预览

Chromium 56（包括V8 v5.6）将发布WebAssembly浏览器预览版。请参考[专门的博客文章](/blog/webassembly-browser-preview)了解更多信息。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.6 -t branch-heads/5.6`以试验 V8 v5.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
