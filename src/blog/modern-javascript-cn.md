***

标题： “ES2015， ES2016， 及以后”
作者： 'the V8 team， ECMAScript Enthusiasts'
日期： 2016-04-29 13：33：37
标签：

*   ECMAScript
    描述： “V8 v5.2 支持 ES2015 和 ES2016！

***

V8 团队非常重视 JavaScript 向一种越来越富有表现力和定义明确的语言的发展，这使得编写快速、安全和正确的 Web 应用程序变得容易。2015年6月，[ES2015规格](https://www.ecma-international.org/ecma-262/6.0/)由 TC39 标准委员会批准，使其成为 JavaScript 语言的最大单次更新。新功能包括[类](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes),[箭头函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions),[承诺](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise),[迭代器/生成器](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators),[代理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy),[知名符号](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Well-known_symbols)，以及额外的句法糖。TC39 还提高了新规范的节奏，并发布了[ES2016候选草案](https://tc39.es/ecma262/2016/)2016年2月，将于今年夏天批准。虽然由于发布周期较短，不像ES2015更新那样广泛，但ES2016特别引入了[幂运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Arithmetic_Operators#Exponentiation)和[`Array.prototype.includes`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/includes).

今天，我们达到了一个重要的里程碑：**V8 支持 ES2015 和 ES2016**.您现在可以在 Chrome Canary 中使用新的语言功能，默认情况下，它们将在 Chrome 52 中提供。

考虑到不断发展的规范的性质，各种类型的一致性测试之间的差异，以及维护Web兼容性的复杂性，当某个版本的ECMAScript被认为是JavaScript引擎完全支持的时，可能很难确定。请继续阅读，了解为什么规范支持比版本号更微妙，为什么适当的尾部调用仍在讨论中，以及还有哪些警告仍在起作用。

## 不断发展的规范

当 TC39 决定更频繁地发布 JavaScript 规范的更新时，该语言的最新版本成为主要的草稿版本。尽管 ECMAScript 规范的版本仍然每年生产和批准，但 V8 实现了最近批准的版本（例如 ES2015）的组合，这些功能与标准化非常接近，可以安全地实现（例如幂运算符和`Array.prototype.includes()`来自ES2016候选草案），以及来自最新草案的错误修复和Web兼容性修订的集合。这种方法的部分理由是，浏览器中的语言实现应该与规范匹配，即使它是需要更新的规范。事实上，实现规范的已批准版本的过程通常会发现构成规范下一版本的许多修复和说明。

![Currently shipping parts of the evolving ECMAScript specification](../_img/modern-javascript/shipped-features.png)

例如，在实现 ES2015 时[正则表达式粘性标志](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky)，V8团队发现ES2015规范的语义打破了许多现有站点（包括所有使用2.x.x版本的流行站点）[XRegExp](https://github.com/slevithan/xregexp)库）。由于兼容性是 Web 的基石，因此来自 V8 和 Safari JavaScriptCore 团队的工程师[提出修正案](https://github.com/tc39/ecma262/pull/511)根据 RegExp 规范来修复破损，这是由 TC39 商定的。该修正案要到 ES2017 才会出现在批准的版本中，但它仍然是 ECMAScript 语言的一部分，我们已经实现了它，以便发布 RegExp 粘性标志。

语言规范的不断完善以及每个版本（包括尚未批准的草案）替换，修改和澄清先前版本的事实使得理解ES2015和ES2016支持背后的复杂性变得棘手。虽然不可能简明扼要地说明，但也许最准确的说法是：*V8 支持遵守“持续维护的未来 ECMAScript 标准草案”*!

## 测量一致性

为了理解这种规范的复杂性，有多种方法可以衡量 JavaScript 引擎与 ECMAScript 标准的兼容性。V8 团队以及其他浏览器供应商使用[Test262 测试套件](https://github.com/tc39/test262)作为符合未来 ECMAScript 标准草案的黄金标准。该测试套件不断更新以匹配规范，并为构成兼容，兼容的JavaScript实现的所有功能和边缘情况提供了16，000个离散功能测试。目前，V8 通过了大约 98% 的 test262，其余 2% 是少数边缘情况和尚未准备好发布的未来 ES 功能。

由于很难浏览大量的 test262 测试，因此存在其他一致性测试，例如[康加克斯兼容性表](http://kangax.github.io/compat-table/ES2015/).Kangax可以很容易地浏览以查看特定功能（如[箭头函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)） 已在给定引擎中实现，但不会像 test262 那样测试所有一致性边缘情况。目前，Chrome Canary在ES2015的Kangax表上得分为98%，在与ES2016相对应的Kangax部分得分为100%（例如，在ESnext选项卡下标记为“2016功能”和“2016 misc”的部分）。

其余2%的Kangax ES2015表测试[正确的尾部调用](http://www.2ality.com/2015/06/tail-call-optimization.html)，该功能已在 V8 中实现，但由于下面详述的出色的开发人员体验问题，在 Chrome Canary 中故意关闭。启用“实验性 JavaScript 功能”标志后，Canary 在 ES2015 的整个 Kangax 表上得分为 100%。

## 正确的尾部呼叫

正确的尾部调用已经实现，但尚未发布，因为对功能的更改是[TC39 目前正在讨论](https://github.com/tc39/proposal-ptc-syntax).ES2015 规定，尾部位置的严格模式函数调用不应导致堆栈溢出。虽然这对某些编程模式来说是一个有用的保证，但当前的语义有两个问题。首先，由于尾部调用消除是隐式的，因此它可以是[程序员难以识别](http://2ality.com/2015/06/tail-call-optimization.html#checking-whether-a-function-call-is-in-a-tail-position)哪些函数实际上处于尾部调用位置。这意味着开发人员可能不会发现其程序中放错位置的尝试尾部调用，直到它们溢出堆栈。其次，实现正确的尾部调用需要从堆栈中省略尾部调用堆栈帧，这会丢失有关执行流的信息。这反过来又会产生两个后果：

1.  这使得在调试期间更难以理解执行是如何到达某个点的，因为堆栈包含不连续性，并且
2.  [`error.stack`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/Stack)包含较少的有关执行流的信息，这些信息可能会破坏收集和分析客户端错误的遥测软件。

实现[阴影堆栈](https://bugs.webkit.org/attachment.cgi?id=274472\&action=review)可以提高调用堆栈的可读性，但 V8 和 DevTools 团队认为，当调试期间显示的堆栈完全确定且始终与实际虚拟机堆栈的真实状态匹配时，调试是最简单、最可靠、最准确的。此外，阴影堆栈在性能方面过于昂贵，无法始终打开。

出于这些原因，V8 团队强烈支持通过特殊语法来表示正确的尾部调用。有一个待定[TC39 提案](https://github.com/tc39/proposal-ptc-syntax)称为句法尾部调用来指定这种行为，由Mozilla和Microsoft的委员会成员共同支持。我们已经按照 ES2015 中的规定实现并分阶段了适当的尾部调用，并开始按照新提案中的规定实现语法尾部调用。V8 团队计划在下次 TC39 会议上解决此问题，然后再默认提供隐式正确的尾部调用或语法尾部调用。在此期间，您可以使用 V8 标志测试每个版本`--harmony-tailcalls`和`--harmony-explicit-tailcalls`.**更新：**这些标志已被删除。

## 模块

ES2015 最令人兴奋的承诺之一是支持 JavaScript 模块将应用程序的不同部分组织和分离到命名空间中。ES2015 指定[`import`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import)和[`export`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export)模块的声明，但不是模块如何加载到 JavaScript 程序中。在浏览器中，加载行为最近通过以下方式指定：[`<script type="module">`](https://blog.whatwg.org/js-modules).尽管需要额外的标准化工作来指定高级动态模块加载 API，但 Chromium 对模块脚本标记的支持已经[开发中](https://groups.google.com/a/chromium.org/d/msg/blink-dev/uba6pMr-jec/tXdg6YYPBAAJ).您可以跟踪[启动错误](https://bugs.chromium.org/p/v8/issues/detail?id=1569)并在[whatwg/loader](https://github.com/whatwg/loader)存储 库。

## ESnext及其他

将来，开发人员可以期望 ECMAScript 更新以更小、更频繁的更新和更短的实现周期进行更新。V8团队已经在努力推出即将推出的功能，例如[`async`/`await`](https://github.com/tc39/ecmascript-asyncawait)关键字[`Object.values`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/values)/[`Object.entries`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries),[`String.prototype.{padStart,padEnd}`](http://tc39.es/proposal-string-pad-start-end/)和[正则表达式查找隐藏](/blog/regexp-lookbehind-assertions)到运行时。请返回查看有关我们针对现有 ES2015 和 ES2016+ 功能的 ESnext 实施进度和性能优化的更多更新。

我们努力继续发展 JavaScript，并在早期实现新功能、确保现有 Web 的兼容性和稳定性以及围绕设计问题提供 TC39 实现反馈之间取得适当的平衡。我们期待看到开发人员使用这些新功能构建的令人难以置信的体验。
