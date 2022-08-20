***

标题： 'JavaScript 代码覆盖率'
作者： '雅各布·格鲁伯 （[@schuay](https://twitter.com/schuay))'
化身：

*   “雅各布-格鲁伯”
    日期： 2017-12-13 13：33：37
    标签：
*   内部
    描述： 'V8 现在具有对 JavaScript 代码覆盖率的本机支持。工具现在可以访问 V8 的覆盖范围信息，而无需检测代码！
    推文：“940879905079873536”

***

代码覆盖率提供有关应用程序的某些部分是否执行以及执行频率（可选）的信息。它通常用于确定测试套件执行特定代码库的彻底程度。

## 为什么它有用？

作为 JavaScript 开发人员，您可能经常发现自己处于代码覆盖率可能有用的情况。例如：

*   对测试套件的质量感兴趣？重构大型遗留项目？代码覆盖率可以准确地显示代码库的哪些部分被覆盖。
*   想要快速了解是否到达了代码库的特定部分？而不是使用`console.log`为`printf`-样式调试或手动单步执行代码，代码覆盖率可以显示有关应用程序的哪些部分已执行的实时信息。
*   或者，也许您正在优化速度，并想知道要关注哪些位置？执行计数可以指出热函数和循环。

## V8 中的 JavaScript 代码覆盖率

今年早些时候，我们在 V8 中添加了对 JavaScript 代码覆盖率的本机支持。版本 5.9 中的初始版本提供了函数粒度的覆盖范围（显示已执行的函数），后来扩展到支持 v6.2 中块粒度的覆盖范围（同样，但对于单个表达式）。

![Function granularity (left) and block granularity (right)](/\_img/javascript-code-coverage/function-vs-block.png)

### 对于 JavaScript 开发人员

目前有两种主要方法可以访问覆盖范围信息。对于JavaScript开发人员，Chrome DevTools的[“覆盖范围”选项卡](https://developers.google.com/web/updates/2017/04/devtools-release-notes#coverage)公开 JS（和 CSS）覆盖率，并在“源”面板中突出显示死代码。

![Block coverage in the DevTools Coverage pane. Covered lines are highlighted in green, uncovered in red.](/\_img/javascript-code-coverage/block-coverage.png)

由于[本杰明·科](https://twitter.com/BenjaminCoe)，还有[持续](https://github.com/bcoe/c8)致力于将V8的代码覆盖率信息集成到流行的[伊斯坦布尔.js](https://istanbul.js.org/)代码覆盖率工具。

![An Istanbul.js report based on V8 coverage data.](/\_img/javascript-code-coverage/istanbul.png)

### 对于嵌入器

嵌入者和框架作者可以直接挂接到 Inspector API，以获得更大的灵活性。V8 提供两种不同的覆盖模式：

1.  *尽力而为的保障*收集覆盖率信息，对运行时性能的影响最小，但可能会丢失有关垃圾回收 （GC） 函数的数据。

2.  *精确覆盖*确保GC不会丢失任何数据，用户可以选择接收执行计数而不是二进制覆盖信息;但性能可能会受到开销增加的影响（有关更多详细信息，请参阅下一节）。可以在功能或块粒度上收集精确的覆盖范围。

用于精确覆盖的检查器 API 如下所示：

*   [`Profiler.startPreciseCoverage(callCount, detailed)`](https://chromedevtools.github.io/devtools-protocol/tot/Profiler/#method-startPreciseCoverage)启用覆盖率收集，可以选择使用调用计数（与二进制覆盖率）和块粒度（与函数粒度）;

*   [`Profiler.takePreciseCoverage()`](https://chromedevtools.github.io/devtools-protocol/tot/Profiler/#method-takePreciseCoverage)将收集的覆盖范围信息作为源范围列表以及关联的执行计数返回;和

*   [`Profiler.stopPreciseCoverage()`](https://chromedevtools.github.io/devtools-protocol/tot/Profiler/#method-stopPreciseCoverage)禁用收集并释放相关数据结构。

通过检查器协议进行的对话可能如下所示：

```json
// The embedder directs V8 to begin collecting precise coverage.
{ "id": 26, "method": "Profiler.startPreciseCoverage",
            "params": { "callCount": false, "detailed": true }}
// Embedder requests coverage data (delta since last request).
{ "id": 32, "method":"Profiler.takePreciseCoverage" }
// The reply contains collection of nested source ranges.
{ "id": 32, "result": { "result": [{
  "functions": [
    {
      "functionName": "fib",
      "isBlockCoverage": true,    // Block granularity.
      "ranges": [ // An array of nested ranges.
        {
          "startOffset": 50,  // Byte offset, inclusive.
          "endOffset": 224,   // Byte offset, exclusive.
          "count": 1
        }, {
          "startOffset": 97,
          "endOffset": 107,
          "count": 0
        }, {
          "startOffset": 134,
          "endOffset": 144,
          "count": 0
        }, {
          "startOffset": 192,
          "endOffset": 223,
          "count": 0
        },
      ]},
      "scriptId": "199",
      "url": "file:///coverage-fib.html"
    }
  ]
}}

// Finally, the embedder directs V8 to end collection and
// free related data structures.
{"id":37,"method":"Profiler.stopPreciseCoverage"}
```

同样，可以使用以下命令检索尽力而为的覆盖范围[`Profiler.getBestEffortCoverage()`](https://chromedevtools.github.io/devtools-protocol/tot/Profiler/#method-getBestEffortCoverage).

## 幕后花絮

如上一节所述，V8 支持两种主要的代码覆盖模式：尽力而为和精确覆盖。请继续阅读，了解其实施的概述。

### 尽力而为的保障

最佳努力和精确的覆盖模式都大量重用了其他V8机制，其中第一种称为*调用计数器*.每次通过 V8 调用函数时[点火](/blog/ignition-interpreter)口译员，我们[递增调用计数器](https://cs.chromium.org/chromium/src/v8/src/builtins/x64/builtins-x64.cc?l=917\&rcl=fc33dfbebfb1cb800d490af97bf1019e9d66be33)在函数的[反馈向量](http://slides.com/ripsawridge/deck).当函数稍后变热并通过优化编译器进行分层时，此计数器用于帮助指导内联有关要内联哪些函数的决策;现在，我们也依靠它来报告代码覆盖率。

第二种重用机制确定函数的源范围。报告代码覆盖率时，调用计数需要绑定到源文件中的关联范围。例如，在下面的示例中，我们不仅需要报告该函数`f`已经执行了一次，但也`f`的源范围从第 1 行开始，到第 3 行结束。

```js
function f() {
  console.log('Hello World');
}

f();
```

我们再次很幸运，能够在 V8 中重用现有信息。函数已经知道它们在源代码中的开始和结束位置，因为[`Function.prototype.toString`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/toString)，这需要知道函数在源文件中的位置才能提取相应的子字符串。

在收集尽力而为的覆盖范围时，这两种机制只是联系在一起：首先，我们通过遍历整个堆来找到所有实时功能。对于每个看到的函数，我们报告调用计数（存储在反馈向量上，我们可以从函数到达）和源范围（方便地存储在函数本身上）。

请注意，由于无论是否启用覆盖率，都会保留调用计数，因此尽力而为的覆盖率不会引入任何运行时开销。它也不使用专用数据结构，因此不需要显式启用或禁用。

那么，为什么这种模式被称为尽力而为，它的局限性是什么呢？超出范围的函数可能会被垃圾回收器释放。这意味着相关的调用计数丢失，实际上我们完全忘记了这些函数曾经存在过。因此，“尽最大努力”：即使我们尽力而为，收集的覆盖范围信息也可能不完整。

### 精确覆盖（功能粒度）

与尽力而为模式相比，精确覆盖可确保提供的覆盖范围信息完整。为了实现这一点，一旦启用了精确覆盖，我们将所有反馈向量添加到 V8 的根引用集，防止 GC 收集它们。虽然这确保了不会丢失任何信息，但它通过人为地保持对象处于活动状态来增加内存消耗。

精确的覆盖模式还可以提供执行计数。这为精确的覆盖实现增加了另一个皱纹。回想一下，每次通过 V8 的解释器调用函数时，调用计数器都会递增，并且该函数可以在变热后进行分层和优化。但是，优化的函数不再增加其调用计数器，因此必须禁用优化编译器才能使其报告的执行计数保持准确。

### 精确覆盖（块粒度）

块粒度覆盖率必须报告正确的覆盖率，直至单个表达式的级别。例如，在下面的代码段中，块覆盖率可以检测到`else`条件表达式的分支`: c`永远不会执行，而函数粒度覆盖率只会知道函数`f`（完整版）已涵盖。

```js
function f(a) {
  return a ? b : c;
}

f(true);
```

您可能还记得，在前面的章节中，我们已经在 V8 中提供了现成的函数调用计数和源范围。不幸的是，对于块覆盖率来说，情况并非如此，我们必须实现新的机制来收集执行计数及其相应的源范围。

第一个方面是源代码范围：假设我们有一个特定块的执行计数，我们如何将它们映射到源代码的一部分？为此，我们需要在解析源文件时收集相关仓位。在块覆盖之前，V8已经在一定程度上做到了这一点。一个例子是函数范围的集合，由于`Function.prototype.toString`如上所述。另一个是源位置用于构造 Error 对象的回溯。但这些都不足以支持区块覆盖;前者仅适用于函数，而后者仅存储位置（例如`if`令牌`if`-`else`语句），而不是源范围。

因此，我们不得不扩展解析器以收集源范围。为了演示，请考虑`if`-`else`陈述：

```js
if (cond) {
  /* Then branch. */
} else {
  /* Else branch. */
}
```

启用区块覆盖后，我们[收集](https://cs.chromium.org/chromium/src/v8/src/parsing/parser-base.h?l=5199\&rcl=cd23cae9edc134ecfe16a4868266dcf5ec432cbf)的源范围`then`和`else`分支，并将它们与解析的`IfStatement`AST 节点。对于其他相关的语言构造也是如此。

在解析期间收集源范围集合后，第二个方面是在运行时跟踪执行计数。这是由[插入](https://cs.chromium.org/chromium/src/v8/src/interpreter/control-flow-builders.cc?l=207\&rcl=cd23cae9edc134ecfe16a4868266dcf5ec432cbf)一个新的专用`IncBlockCounter`在生成的字节码数组中的战略位置的字节码。在运行时，`IncBlockCounter`字节码处理程序简单[增量](https://cs.chromium.org/chromium/src/v8/src/runtime/runtime-debug.cc?l=2012\&rcl=cd23cae9edc134ecfe16a4868266dcf5ec432cbf)相应的计数器（可通过函数对象访问）。

在上面的示例中`if`-`else`语句，这样的字节码将插入到三个位置：紧挨着`then`分支，在主体之前`else`分支，紧接在`if`-`else`语句（由于分支内可能存在非本地控制，因此需要此类继续计数器）。

最后，报告块粒度覆盖率的工作方式类似于功能粒度报告。但是，除了调用计数（来自反馈向量）之外，我们现在还报告了*有趣*源范围及其块计数（存储在挂起函数的辅助数据结构上）。

如果您想了解有关 V8 中代码覆盖率背后的技术细节的更多信息，请参阅[覆盖](https://goo.gl/WibgXw)和[块覆盖](https://goo.gl/hSJhXn)设计文档。

## 结论

我们希望您喜欢这个关于 V8 本机代码覆盖率支持的简要介绍。请尝试一下，不要犹豫，让我们知道什么适合您，什么不适合您。在推特上打个招呼（[@schuay](https://twitter.com/schuay)和[@hashseed](https://twitter.com/hashseed)） 或提交错误[crbug.com/v8/new](https://crbug.com/v8/new).

V8中的覆盖支持一直是团队的努力，感谢所有做出贡献的人：Benjamin Coe，Jakob Gruber，Yang Guo，Marja Hölttä，Andrey Kosyakov，Alexey Kozyatinksiy，Ross McIlroy，Ali Sheikh，Michael Starzinger。谢谢！
