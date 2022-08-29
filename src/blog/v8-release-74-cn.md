***

标题： 'V8 版本 v7.4'
作者： 'Georg Neis'
日期： 2019-03-22 16：30：42
标签：

*   释放
    描述： 'V8 v7.4 具有 WebAssembly 线程/原子、私有类字段、性能和内存改进等等！
    推文：“1109094755936489472”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.4)，直到几周后与Chrome 74 Stable一起发布之前，它一直处于测试阶段。V8 v7.4 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 无JIT V8

V8 现在支持*JavaScript*执行而不在运行时分配可执行内存。有关此功能的深入信息，请参见[专门的博客文章](/blog/jitless).

## WebAssembly Threads/Atomics 已发货

WebAssembly Threads/Atomics 现在在非 Android 操作系统上启用。至此结束[我们在 V8 v7.0 中启用的源试用版/预览版](/blog/v8-release-70#a-preview-of-webassembly-threads).Web 基础知识文章对此进行了解释[如何将 WebAssembly Atomics 与 Emscripten 一起使用](https://developers.google.com/web/updates/2018/10/wasm-threads).

这通过WebAssembly解锁了用户机器上多个内核的使用，从而在Web上实现了新的，计算繁重的用例。

## 性能

### 参数不匹配的更快呼叫

在JavaScript中，调用参数太少或太多（即传递的参数少于或多于声明的形式参数）的函数是完全有效的。前者称为*申请不足*，后者称为*过度应用*.如果应用不足，则分配其余形式参数`undefined`，而在过度应用的情况下，多余的参数将被忽略。

但是，JavaScript 函数仍然可以通过[`arguments`对象](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments)，通过使用[休息参数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters)，甚至通过使用非标准[`Function.prototype.arguments`财产](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/arguments)上[草率模式](https://developer.mozilla.org/en-US/docs/Glossary/Sloppy_mode)功能。因此，JavaScript 引擎必须提供一种获取实际参数的方法。在 V8 中，这是通过一种称为*参数适应*，它在应用不足或过度应用的情况下提供实际参数。不幸的是，参数适应是以性能为代价的，并且在现代前端和中间件框架中通常需要（即许多具有可选参数或可变参数列表的API）。

在某些情况下，引擎知道参数适应不是必需的，因为无法观察到实际参数，即当被调用方是严格模式函数时，并且两者都不使用`arguments`也不是 rest 参数。在这些情况下，V8 现在完全跳过了参数调整，从而将调用开销降低多达**60%**.

![Performance impact of skipping arguments adaption, as measured through a micro-benchmark.](../_img/v8-release-74/argument-mismatch-performance.svg)

该图显示，即使参数不匹配（假设被调用方无法观察实际参数），也不再有开销。有关更多详细信息，请参阅[设计文档](https://bit.ly/v8-faster-calls-with-arguments-mismatch).

### 改进的本机访问器性能

Angular 团队[发现](https://mhevery.github.io/perf-tests/DOM-megamorphic.html)直接通过各自的访问器（即 DOM 属性访问器）调用本机访问器`get`Chrome 中的函数明显慢于[单态](https://en.wikipedia.org/wiki/Inline_caching#Monomorphic_inline_caching)甚至[巨态](https://en.wikipedia.org/wiki/Inline_caching#Megamorphic_inline_caching)属性访问。这是由于在 V8 中采用慢速路径来调用 DOM 访问器，通过[`Function#call()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)，而不是已经存在用于属性访问的快速路径。

![](../_img/v8-release-74/native-accessor-performance.svg)

我们设法提高了调用本机访问器的性能，使其比巨型属性访问快得多。有关更多背景信息，请参阅[V8 问题 #8820](https://bugs.chromium.org/p/v8/issues/detail?id=8820).

### 解析器性能

在 Chrome 中，足够大的脚本在下载时会在工作线程上进行“流式处理”解析。在此版本中，我们发现并修复了源流使用的自定义 UTF-8 解码的性能问题，从而使流式解析平均提高 8%。

我们在 V8 的 preparser 中发现了另一个问题，它最常在工作线程上运行：属性名称被不必要地重复数据删除。删除此重复数据删除可将流式分析器再提高 10.5%。这还缩短了未流式传输的脚本（如小脚本和内联脚本）的主线程解析时间。

![Each drop in the above chart represents one of the performance improvements in the streaming parser.](../_img/v8-release-74/parser-performance.jpg)

## 记忆

### 字节码刷新

从 JavaScript 源代码编译的字节码占用了 V8 堆空间的很大一部分，通常约为 15%，包括相关的元数据。有许多函数仅在初始化期间执行，或者在编译后很少使用。

为了减少 V8 的内存开销，我们实现了对在垃圾回收期间从函数中刷新已编译字节码的支持（如果它们最近未执行）。为了实现这一点，我们跟踪函数字节码的期限，在垃圾回收期间增加期限，并在执行函数时将其重置为零。任何超过老化阈值的字节码都有资格被下一个垃圾回收收集，如果将来再次执行，该函数将重置为懒惰地重新编译其字节码。

我们对字节码刷新的实验表明，它为Chrome用户节省了大量内存，将V8堆中的内存量减少了5-15%，同时不会降低性能或显着增加编译JavaScript代码所花费的CPU时间。

![](../_img/v8-release-74/bytecode-flushing.svg)

### 字节码死基本块消除

Ignition字节码编译器试图避免生成它知道已经死亡的代码，例如`return`或`break`陈述：

```js
return;
deadCall(); // skipped
```

但是，以前，这是为了终止语句列表中的语句而机会性的，因此它没有考虑其他优化，例如已知为真的快捷方式条件：

```js
if (2.2) return;
deadCall(); // not skipped
```

我们试图在 V8 v7.3 中解决这个问题，但仍然在每语句级别上，当控制流变得更加复杂时，这将不起作用，例如

```js
do {
  if (2.2) return;
  break;
} while (true);
deadCall(); // not skipped
```

这`deadCall()`上面将位于一个新的基本块的开头，在每个语句级别上，该块可以作为目标访问`break`循环中的语句。

在 V8 v7.4 中，我们允许整个基本块失效，如果没有`Jump`字节码（Ignition的主控制流原语）指的是它们。在上面的示例中，`break`未发出，这意味着循环没有`break`语句。所以，基本块从`deadCall()`没有参考跳跃，因此也被认为是死的。虽然我们预计这不会对用户代码产生很大的影响，但它对于简化各种脱糖特别有用，例如生成器，`for-of`和`try-catch`，特别是删除了一类错误，在这些错误中，基本块可以在实现过程中“复活”复杂的语句。

## JavaScript 语言特性

### 私有类字段

V8 v7.2 添加了对公共类字段语法的支持。类字段通过避免构造函数仅用于定义实例属性来简化类语法。从 V8 v7.4 开始，您可以通过在字段前面附加一个字段来将其标记为私有`#`前缀。

```js
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Getting the current value!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}
```

与公共字段不同，私有字段在类体外部不可访问：

```js
const counter = new IncreasingCounter();
counter.#count;
// → SyntaxError
counter.#count = 42;
// → SyntaxError
```

欲了解更多信息，请阅读我们的[公共和私人类字段的解释器](/features/class-fields).

### `Intl.Locale`

JavaScript 应用程序通常使用字符串，例如`'en-US'`或`'de-CH'`以标识区域设置。`Intl.Locale`提供了更强大的机制来处理区域设置，并可以轻松提取特定于区域设置的首选项，如语言、日历、编号系统、小时周期等。

```js
const locale = new Intl.Locale('es-419-u-hc-h12', {
  calendar: 'gregory'
});
locale.language;
// → 'es'
locale.calendar;
// → 'gregory'
locale.hourCycle;
// → 'h12'
locale.region;
// → '419'
locale.toString();
// → 'es-419-u-ca-gregory-hc-h12'
```

### 哈希邦语法

JavaScript 程序现在可以从`#!`，一个所谓的[哈希邦](https://github.com/tc39/proposal-hashbang).哈希邦后面的行的其余部分被视为单行注释。这与命令行 JavaScript 主机（如 Node.js）中的实际用法相匹配。下面现在是一个语法上有效的 JavaScript 程序：

```js
#!/usr/bin/env node
console.log(42);
```

## V8 接口

请使用`git log branch-heads/7.3..branch-heads/7.4 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.4 -t branch-heads/7.4`以试验 V8 v7.4 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
