***

标题： “极快解析，第 2 部分：懒惰解析”
作者： 'Toon Verwaest （[@tverwaes](https://twitter.com/tverwaes)） 和 Marja Hölttä （[@marjakh](https://twitter.com/marjakh)），稀疏解析器
化身：

*   'toon-verwaest'
*   'marja-holtta'
    日期： 2019-04-15 17：03：37
    标签：
*   内部
*   解析
    推文：“1117807107972243456”
    描述： '这是我们系列文章的第二部分，解释了 V8 如何尽可能快地解析 JavaScript。

***

这是我们系列的第二部分，解释 V8 如何尽可能快地解析 JavaScript。第一部分解释了我们如何制作V8的[扫描器](/blog/scanner)快。

解析是将源代码转换为中间表示以供编译器使用的步骤（在 V8 中，字节码编译器[点火](/blog/ignition-interpreter)).解析和编译发生在网页启动的关键路径上，并且在启动期间并不需要所有附带到浏览器的功能。尽管开发人员可以使用异步和延迟脚本延迟此类代码，但这并不总是可行的。此外，许多网页附带的代码仅由某些功能使用，在页面的任何单独运行期间，用户可能根本无法访问这些功能。

急于不必要地编译代码会带来真正的资源成本：

*   CPU 周期用于创建代码，从而延迟启动实际需要的代码的可用性。
*   代码对象占用内存，至少直到[字节码刷新](/blog/v8-release-74#bytecode-flushing)确定当前不需要该代码，并允许对其进行垃圾回收。
*   顶级脚本完成执行时编译的代码最终会缓存在磁盘上，从而占用磁盘空间。

由于这些原因，所有主要浏览器都实现了*惰性解析*.解析器可以决定“预解析”它遇到的函数，而不是为每个函数生成抽象语法树 （AST），然后将其编译为字节码。它通过切换到[准备者](https://cs.chromium.org/chromium/src/v8/src/parsing/preparser.h?l=921\&rcl=e3b2feb3aade83c02e4bd2fa46965a69215cd821)，解析器的副本，它执行所需的最低限度，以便能够跳过该函数。preparser 验证它跳过的函数在语法上是否有效，并生成正确编译外部函数所需的所有信息。以后调用预准备函数时，将根据需要对其进行完全分析和编译。

## 变量分配

使预解析复杂化的主要因素是变量分配。

出于性能原因，函数激活在计算机堆栈上进行管理。例如，如果一个函数`g`调用函数`f`带参数`1`和`2`:

```js
function f(a, b) {
  const c = a + b;
  return c;
}

function g() {
  return f(1, 2);
  // The return instruction pointer of `f` now points here
  // (because when `f` `return`s, it returns here).
}
```

首先是接收器（即`this`的值`f`，即`globalThis`因为它是一个草率的函数调用）被推送到堆栈上，然后是被调用的函数`f`.然后参数`1`和`2`被推到堆栈上。此时函数`f`被调用。要执行调用，我们首先保存`g`在堆栈上：“返回指令指针”（`rip`;我们需要返回哪些代码）`f`以及“帧指针”（`fp`;返回时堆栈应是什么样子）。然后我们进入`f`，为局部变量分配空间`c`，以及它可能需要的任何临时空间。这可确保函数使用的任何数据在函数激活超出范围时消失：它只是从堆栈中弹出。

![Stack layout of a call to function f with arguments a, b, and local variable c allocated on the stack.](../_img/preparser/stack-1.svg)

此设置的问题在于函数可以引用在外部函数中声明的变量。内部函数的寿命可以超过创建它们的激活时间：

```js
function make_f(d) { // ← declaration of `d`
  return function inner(a, b) {
    const c = a + b + d; // ← reference to `d`
    return c;
  };
}

const f = make_f(10);

function g() {
  return f(1, 2);
}
```

在上面的示例中，引用来自`inner`到局部变量`d`声明于`make_f`在以下时间后进行评估`make_f`已返回。为了实现这一点，具有词法闭包的语言的 VM 在称为“上下文”的结构中分配从堆上的内部函数引用的变量。

![Stack layout of a call to make_f with the argument copied to a context allocated on the heap for later use by inner that captures d.](../_img/preparser/stack-2.svg)

这意味着对于函数中声明的每个变量，我们需要知道内部函数是否引用了该变量，以便我们可以决定是在堆栈上还是在堆分配的上下文中分配变量。当我们计算函数文本时，我们分配一个闭包，该闭包既指向函数的代码，也指向当前上下文：包含可能需要访问的变量值的对象。

长话短说，我们确实需要在准备器中至少跟踪变量引用。

但是，如果我们只跟踪引用，我们会高估引用的变量。在外部函数中声明的变量可以通过内部函数中的重新声明来隐藏，使来自该内部函数的引用以内部声明为目标，而不是外部声明。如果我们在上下文中无条件地分配外部变量，性能将受到影响。因此，为了使变量分配能够正确处理准备工作，我们需要确保准备的函数正确跟踪变量引用和声明。

顶级代码是此规则的例外。脚本的顶层始终是堆分配的，因为变量在脚本之间是可见的。接近一个工作良好的架构的一种简单方法是简单地运行reader，而无需变量跟踪来快速解析顶级函数;并对内部函数使用完整的解析器，但跳过编译它们。这比准备工作更昂贵，因为我们不必要地构建了整个AST，但它可以让我们启动并运行。这正是V8在V8 v6.3 / Chrome 63之前所做的。

## 向准备者传授变量

在 preparser 中跟踪变量声明和引用很复杂，因为在 JavaScript 中，从一开始就不清楚分部表达式的含义是什么。例如，假设我们有一个函数`f`带参数`d`，它具有内部功能`g`具有看起来可能引用的表达式`d`.

```js
function f(d) {
  function g() {
    const a = ({ d }
```

它确实可能最终引用`d`，因为我们看到的标记是解构赋值表达式的一部分。

```js
function f(d) {
  function g() {
    const a = ({ d } = { d: 42 });
    return a;
  }
  return g;
}
```

它也可能最终成为具有解构参数的箭头函数。`d`，在这种情况下`d`在`f`未被引用`g`.

```js
function f(d) {
  function g() {
    const a = ({ d }) => d;
    return a;
  }
  return [d, g];
}
```

最初，我们的 preparser 是作为解析器的独立副本实现的，没有太多的共享，这导致两个解析器随着时间的推移而发散。通过将解析器和准备器重写为基于`ParserBase`实现[奇怪的重复模板模式](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)，我们设法最大限度地提高了共享，同时保持了单独副本的性能优势。这大大简化了向准备程序添加完整变量跟踪的过程，因为实现的很大一部分可以在解析器和准备程序之间共享。

实际上，即使对于顶级函数，忽略变量声明和引用也是不正确的。ECMAScript 规范要求在首次解析脚本时检测各种类型的变量冲突。例如，如果一个变量在同一范围内两次声明为词法变量，则被视为[早`SyntaxError`](https://tc39.es/ecma262/#early-error).由于我们的 preparser 只是跳过了变量声明，因此在 preparse 期间会错误地允许代码。当时我们认为性能的胜利保证了规范的违反。但是，现在 preparser 可以正确跟踪变量，因此我们消除了这整类与变量分辨率相关的规范违规，而没有显著的性能成本。

## 跳过内部函数

如前所述，当第一次调用预准备函数时，我们对其进行完全解析并将生成的AST编译为字节码。

```js
// This is the top-level scope.
function outer() {
  // preparsed
  function inner() {
    // preparsed
  }
}

outer(); // Fully parses and compiles `outer`, but not `inner`.
```

该函数直接指向外部上下文，该上下文包含需要可用于内部函数的变量声明的值。为了允许函数的延迟编译（并支持调试器），上下文指向一个名为[`ScopeInfo`](https://cs.chromium.org/chromium/src/v8/src/objects/scope-info.h?rcl=ce2242080787636827dd629ed5ee4e11a4368b9e\&l=36).`ScopeInfo`对象描述上下文中列出的变量。这意味着在编译内部函数时，我们可以计算变量在上下文链中的位置。

但是，为了计算惰性编译函数本身是否需要上下文，我们需要再次执行范围解析：我们需要知道嵌套在惰性编译函数中的函数是否引用惰性函数声明的变量。我们可以通过重新准备这些函数来解决这个问题。这正是V8在V8 v6.3 / Chrome 63之前所做的。然而，这在性能方面并不理想，因为它使源代码大小和解析成本之间的关系非线性：我们将准备函数的次数与嵌套的次数一样多。除了动态程序的自然嵌套之外，JavaScript打包者通常将代码包装在”[立即调用的函数表达式](https://en.wikipedia.org/wiki/Immediately_invoked_function_expression)“（IIFEs），使大多数JavaScript程序具有多个嵌套层。

![Each reparse adds at least the cost of parsing the function.](../_img/preparser/parse-complexity-before.svg)

为了避免非线性性能开销，即使在准备过程中，我们也会执行全范围分辨率。我们存储了足够的元数据，以便以后可以简单地*跳*内部功能，而不必重新准备它们。一种方法是存储内部函数引用的变量名称。这存储成本很高，并且要求我们仍然需要重复工作：我们已经在准备过程中执行了可变分辨率。

相反，我们序列化了变量作为每个变量的标志密集数组分配的位置。当我们对函数进行惰性解析时，变量将按照与准备器相同的顺序重新创建，我们可以简单地将元数据应用于变量。现在该函数已编译，不再需要变量分配元数据，并且可以进行垃圾回收。由于我们只需要实际包含内部函数的函数的元数据，因此所有函数中的很大一部分甚至不需要此元数据，从而显着降低了内存开销。

![By keeping track of metadata for preparsed functions we can completely skip inner functions.](../_img/preparser/parse-complexity-after.svg)

跳过内部函数对性能的影响是非线性的，就像重新准备内部函数的开销一样。有些站点将其所有功能提升到顶级范围。由于它们的嵌套级别始终为 0，因此开销始终为 0。但是，许多现代网站实际上确实深嵌套了功能。在这些网站上，当此功能在V8 v6.3 / Chrome 63中推出时，我们看到了显着的改进。主要优点是，现在代码嵌套的深度不再重要：任何函数最多准备一次，完全解析一次\[^1]。

![Main thread and off-the-main-thread parse time, before and after launching the “skipping inner functions” optimization.](../_img/preparser/skipping-inner-functions.svg)

\[^1]：出于内存原因，V8[刷新字节码](/blog/v8-release-74#bytecode-flushing)当它被闲置一段时间时。如果以后再次需要该代码，我们将重新解析并再次编译它。由于我们允许变量元数据在编译期间失效，因此在延迟重新编译时会导致内部函数的重新分析。在这一点上，我们为其内部函数重新创建元数据，因此我们不需要再次重新准备其内部函数的内部函数。

## 可能调用的函数表达式 { #pife }

如前所述，打包程序通常通过将模块代码包装在他们立即调用的闭包中，将多个模块组合在单个文件中。这为模块提供了隔离，允许它们像脚本中唯一的代码一样运行。这些函数本质上是嵌套脚本;脚本执行时会立即调用这些函数。包装工通常发货*立即调用的函数表达式*（IIFEs;发音为“iffies”）作为括号中的函数：`(function(){…})()`.

由于在脚本执行期间立即需要这些函数，因此准备此类函数并不理想。在脚本的顶级执行期间，我们立即需要编译函数，并且我们完全解析和编译函数。这意味着我们之前为尝试加快启动速度而进行的更快的解析肯定会成为启动过程中不必要的额外成本。

你可能会问，你为什么不简单地编译被调用的函数呢？虽然开发人员通常可以直接注意到何时调用函数，但对于解析器而言，情况并非如此。解析器需要做出决定 - 甚至在它开始解析函数之前！— 它是否想热切地编译函数或推迟编译。语法中的歧义使得很难简单地快速扫描到函数的末尾，并且成本很快类似于常规准备的成本。

出于这个原因，V8有两个简单的模式，它识别为*可能调用的函数表达式*（PIFE;发音为“piffies”），它急切地解析和编译一个函数：

*   如果函数是带括号的函数表达式，即`(function(){…})`，我们假设它将被调用。一旦我们看到这种模式的开始，我们就会做出这个假设，即`(function`.
*   由于V8 v5.7 / Chrome 57我们也检测到模式`!function(){…}(),function(){…}(),function(){…}()`生成者[UglifyJS](https://github.com/mishoo/UglifyJS2).一旦我们看到，此检测就会启动`!function`或`,function`如果它紧跟在 PIFE 之后。

由于 V8 急切地编译了 PIFE，因此它们可以用作[配置文件定向反馈](https://en.wikipedia.org/wiki/Profile-guided_optimization)\[^2]，通知浏览器启动需要哪些功能。

在 V8 仍然重新解析内部函数的时候，一些开发人员注意到 JS 解析对启动的影响非常大。套餐[`optimize-js`](https://github.com/nolanlawson/optimize-js)将函数转换为基于静态启发式的 PIFE。在创建软件包时，这对 V8 上的负载性能产生了巨大的影响。我们通过运行由`optimize-js`在 V8 v6.1 上，只查看缩小的脚本。

![Eagerly parsing and compiling PIFEs results in slightly faster cold and warm startup (first and second page load, measuring total parse + compile + execute times). The benefit is much smaller on V8 v7.5 than it used to be on V8 v6.1 though, due to significant improvements to the parser.](../_img/preparser/eager-parse-compile-pife.svg)

尽管如此，现在我们不再重新解析内部函数，并且由于解析器的速度要快得多，因此通过`optimize-js`大大减少。实际上，v7.5 的默认配置已经比在 v6.1 上运行的优化版本快得多。即使在 v7.5 上，对启动期间所需的代码谨慎使用 PIFE 仍然是有意义的：我们避免了 preparse，因为我们很早就知道需要该函数。

这`optimize-js`基准测试结果并不能完全反映现实世界。脚本是同步加载的，整个解析 + 编译时间计入加载时间。在实际设置中，您可能会使用`<script>`标签。这允许Chrome的预加载器发现脚本。*以前*它被评估，并在不阻塞主线程的情况下下载，解析和编译脚本。我们决定急切编译的所有内容都是从主线程自动编译的，并且应该只计入启动。使用非主线程脚本编译运行可放大使用 PIFE 的影响。

但是，仍然有成本，尤其是内存成本，因此急切地编译所有内容并不是一个好主意：

![Eagerly compiling all JavaScript comes at a significant memory cost.](../_img/preparser/eager-compilation-overhead.svg)

虽然在启动期间所需的函数周围添加括号是一个好主意（例如，基于分析启动），但使用像这样的包`optimize-js`应用简单的静态启发式方法不是一个好主意。例如，它假设在启动期间将调用一个函数，如果它是函数调用的参数。但是，如果这样的函数实现了整个模块，而该模块仅在很久以后才需要，那么您最终会编译太多。过度热切编译对性能不利：没有延迟编译的 V8 会显著减少加载时间。此外，一些好处`optimize-js`来自UglifyJS和其他微型器的问题，这些微缩模型从不是IIFE的PIFE中删除括号，删除了可以应用于例如[通用模块定义](https://github.com/umdjs/umd)-样式模块。这可能是一个微型浏览器应该解决的问题，以便在急切编译PIFE的浏览器上获得最佳性能。

\[^2]：PIFE 也可以被认为是基于配置文件的函数表达式。

## 结论

惰性解析可加快启动速度，并减少交付的代码多于所需代码的应用程序的内存开销。能够在准备器中正确跟踪变量声明和引用对于能够正确（根据规范）和快速地准备是必要的。在准备器中分配变量还允许我们序列化变量分配信息，以便以后在解析器中使用，这样我们就可以避免完全重新准备内部函数，从而避免深度嵌套函数的非线性解析行为。

解析器可以识别的 PIFE 可避免启动期间立即需要的代码的初始准备开销。小心按配置文件使用 PIFE 或由包装工使用，可以提供有用的冷启动减速带。但是，应避免不必要地将函数包装在括号中以触发此启发式方法，因为这会导致更多代码被紧急编译，从而导致启动性能变差并增加内存使用量。
