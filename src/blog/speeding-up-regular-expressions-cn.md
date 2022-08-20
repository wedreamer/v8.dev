***

标题： “加速 V8 正则表达式”
作者：“Jakob Gruber，常规软件工程师”
化身：

*   “雅各布-格鲁伯”
    日期： 2017-01-10 13：33：37
    标签：
*   内部
*   正则表达式
    描述： 'V8 最近将 RegExp 的内置函数从自托管 JavaScript 实现迁移到直接连接到我们基于 TurboFan 的新代码生成架构的实现。

***

这篇博客文章介绍了 V8 最近将 RegExp 的内置函数从自托管 JavaScript 实现迁移到直接连接到我们基于以下特性的新代码生成架构的实现：[涡轮风扇](/blog/v8-release-56).

V8 的 RegExp 实现建立在[Irregexp](https://blog.chromium.org/2009/02/irregexp-google-chromes-new-regexp.html)，它被广泛认为是最快的正则表达式引擎之一。虽然引擎本身封装了低级逻辑以对字符串执行模式匹配，但RegExp原型上的函数例如[`RegExp.prototype.exec`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec)执行向用户公开其功能所需的其他工作。

从历史上看，V8 的各种组件都是用 JavaScript 实现的。直到最近，`regexp.js`一直是其中之一，托管RegExp构造函数的实现，其所有属性以及其原型的属性。

遗憾的是，这种方法有缺点，包括不可预测的性能以及低级功能向C++运行时的昂贵转换。最近在 ES6 中添加了内置的子类化（允许 JavaScript 开发人员提供自己的自定义 RegExp 实现）导致了进一步的 RegExp 性能损失，即使 RegExp 内置的 RegExp 没有被子类化。这些回归无法在自托管 JavaScript 实现中完全解决。

因此，我们决定将 RegExp 实现从 JavaScript 迁移出去。 然而，事实证明，保持性能比预期的要困难得多。最初迁移到完整C++实现的速度明显较慢，仅达到原始实现性能的 70% 左右。 经过一番调查，我们发现了几个原因：

*   [`RegExp.prototype.exec`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec)包含几个对性能极其敏感的区域，最值得注意的是，包括转换到基础 RegExp 引擎，以及 RegExp 结果及其关联的子字符串调用的构造。对于这些，JavaScript实现依赖于高度优化的代码片段，称为“stubs”，这些代码段是用本机汇编语言编写的，或者通过直接挂接到优化编译器管道中来编写的。无法从C++访问这些存根，并且它们的运行时等效项明显变慢。
*   访问 RegExp 等属性`lastIndex`可能很昂贵，可能需要按名称和遍历原型链进行查找。V8 的优化编译器通常可以用更有效的操作自动替换这些访问，而这些情况需要在C++中显式处理。
*   在C++中，对JavaScript对象的引用必须包装在所谓的`Handle`s 为了配合垃圾回收。与普通的 JavaScript 实现相比，句柄管理会产生进一步的开销。

我们的正则表达式迁移的新设计基于[CodeStubAssembler](/blog/csa)，一种允许 V8 开发人员编写独立于平台的代码的机制，这些代码稍后将由相同的后端转换为快速的、特定于平台的代码，该后端也用于新的优化编译器 TurboFan。使用CodeStubAssembler使我们能够解决初始C++实现的所有缺点。存根（例如 RegExp 引擎的入口点）可以从 CodeStubAssembler 轻松调用。虽然快速属性访问仍然需要在所谓的快速路径上显式实现，但这种访问在CodeStubAssembler中非常有效。句柄根本不存在于C++之外。由于实现现在在非常低的水平上运行，我们可以采取进一步的捷径，例如在不需要时跳过昂贵的结果构造。

结果非常积极。我们的得分[大量的正则表达式工作负载](https://github.com/chromium/octane/blob/master/regexp.js)提高了 15%，超过了重新获得我们最近与子类相关的性能损失。微基准标记（图1）显示了全面的改进，从7%到[`RegExp.prototype.exec`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/exec)，高达 102% 用于[`RegExp.prototype[@@split]`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/@@split).

![Figure 1: RegExp speedup broken down by function](/\_img/speeding-up-regular-expressions/perf.png)

那么，作为 JavaScript 开发人员，您如何确保您的正则表达式速度快呢？如果您对挂接到 RegExp 内部组件不感兴趣，请确保既不修改 RegExp 实例，也不修改其原型以获得最佳性能：

```js
const re = /./g;
re.exec('');  // Fast path.
re.new_property = 'slow';
RegExp.prototype.new_property = 'also slow';
re.exec('');  // Slow path.
```

虽然 RegExp 子类化有时可能非常有用，但请注意，子类化 RegExp 实例需要更多的通用处理，因此采用慢速路径：

```js
class SlowRegExp extends RegExp {}
new SlowRegExp(".", "g").exec('');  // Slow path.
```

完整的正则表达式迁移将在 V8 v5.7 中提供。
