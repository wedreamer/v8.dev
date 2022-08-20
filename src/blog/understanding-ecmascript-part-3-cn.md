***

标题： “了解 ECMAScript 规范，第 3 部分”
作者： '[Marja Hölttä](https://twitter.com/marjakh)，投机规格观众
化身：

*   马利亚-霍尔塔
    日期： 2020-04-01
    标签：
*   ECMAScript
*   了解 ECMAScript
    描述： '关于阅读 ECMAScript 规范的教程'
    推文：“1245400717667577857”

***

[所有剧集](/blog/tags/understanding-ecmascript)

在本集中，我们将更深入地了解 ECMAScript 语言及其语法的定义。如果您不熟悉上下文无关语法，现在是查看基础知识的好时机，因为规范使用上下文无关语法来定义语言。看[“制作口译员”中关于上下文无关语法的章节](https://craftinginterpreters.com/representing-code.html#context-free-grammars)对于平易近人的介绍或[维基百科页面](https://en.wikipedia.org/wiki/Context-free_grammar)以获得更数学化的定义。

## ECMAScript 语法

ECMAScript 规范定义了四种语法：

这[词汇语法](https://tc39.es/ecma262/#sec-ecmascript-language-lexical-grammar)描述如何[统一码码位](https://en.wikipedia.org/wiki/Unicode#Architecture_and_terminology)被翻译成一个序列**输入元素**（标记、行终止符、注释、空格）。

这[句法语法](https://tc39.es/ecma262/#sec-syntactic-grammar)定义语法正确的程序如何由标记组成。

这[正则表达式语法](https://tc39.es/ecma262/#sec-patterns)描述如何将 Unicode 码位转换为正则表达式。

这[数字字符串语法](https://tc39.es/ecma262/#sec-tonumber-applied-to-the-string-type)描述如何将字符串转换为数值。

每个语法都被定义为一个上下文无关的语法，由一组作品组成。

语法使用略有不同的符号：句法语法使用`LeftHandSideSymbol :`而词汇语法和正则表达式语法使用`LeftHandSideSymbol ::`和数字字符串语法使用`LeftHandSideSymbol :::`.

接下来，我们将更详细地研究词汇语法和句法语法。

## 词汇语法

该规范将 ECMAScript 源文本定义为一系列 Unicode 码位。例如，变量名称不限于 ASCII 字符，还可以包含其他 Unicode 字符。规范不讨论实际的编码（例如，UTF-8 或 UTF-16）。它假设源代码已经根据其所处的编码转换为一系列 Unicode 码位。

不可能提前标记 ECMAScript 源代码，这使得定义词法语法稍微复杂一些。

例如，我们无法确定是否`/`是除法运算符或正则表达式的开始，而不查看它发生的较大上下文：

```js
const x = 10 / 5;
```

这里`/`是一个`DivPunctuator`.

```js
const r = /foo/;
```

这里第一个`/`是一个开始`RegularExpressionLiteral`.

模板引入了类似的歧义 - 解释<code>}\`</code>取决于它出现在以下上下文中：

```js
const what1 = 'temp';
const what2 = 'late';
const t = `I am a ${ what1 + what2 }`;
```

这里<code>'我是${</code>是`TemplateHead`和<code>}\`</code>是一个`TemplateTail`.

```js
if (0 == 1) {
}`not very useful`;
```

这里`}`是一个`RightBracePunctuator`和<code>\`</code>是一个开始`NoSubstitutionTemplate`.

即使解释`/`和<code>}\`</code>取决于它们的“上下文”——它们在代码句法结构中的位置——我们接下来要描述的语法仍然是上下文无关的。

词法语法使用多个目标符号来区分允许某些输入元素和某些输入元素不允许的上下文。例如，目标符号`InputElementDiv`在以下上下文中使用`/`是一个部门和`/=`是一个部门分配。这[`InputElementDiv`](https://tc39.es/ecma262/#prod-InputElementDiv)生产列出了可以在以下上下文中生产的可能令牌：

```grammar
InputElementDiv ::
  WhiteSpace
  LineTerminator
  Comment
  CommonToken
  DivPunctuator
  RightBracePunctuator
```

在此背景下，遇到`/`产生`DivPunctuator`输入元素。生成`RegularExpressionLiteral`在这里不是一个选项。

另一方面[`InputElementRegExp`](https://tc39.es/ecma262/#prod-InputElementRegExp)是上下文中的目标符号，其中`/`是正则表达式的开始：

```grammar
InputElementRegExp ::
  WhiteSpace
  LineTerminator
  Comment
  CommonToken
  RightBracePunctuator
  RegularExpressionLiteral
```

正如我们从制作中看到的那样，这有可能产生`RegularExpressionLiteral`输入元素，但正在生产`DivPunctuator`是不可能的。

同样，还有另一个目标符号，`InputElementRegExpOrTemplateTail`，用于以下上下文`TemplateMiddle`和`TemplateTail`是允许的，除了`RegularExpressionLiteral`.最后，`InputElementTemplateTail`是仅`TemplateMiddle`和`TemplateTail`是允许的，但`RegularExpressionLiteral`是不允许的。

在实现中，句法语法分析器（“解析器”）可以调用词法语法分析器（“分词器”或“词法分析器”），将目标符号作为参数传递，并要求下一个适合该目标符号的输入元素。

## 句法语法

我们研究了词法语法，它定义了我们如何从Unicode码位构造令牌。句法语法建立在它之上：它定义了语法正确的程序如何由标记组成。

### 示例：允许使用旧标识符

在语法中引入新关键字可能是一个重大更改 — 如果现有代码已经使用关键字作为标识符，该怎么办？

例如，在`await`是一个关键字，有人可能已经编写了以下代码：

```js
function old() {
  var await;
}
```

ECMAScript 语法精心添加了`await`关键字，以便此代码继续工作。在异步函数内部，`await`是一个关键字，所以这不起作用：

```js
async function modern() {
  var await; // Syntax error
}
```

允许`yield`作为非生成器中的标识符，在生成器中不允许它的工作方式类似。

了解如何`await`允许作为标识符，需要理解特定于 ECMAScript 的语法语法表示法。让我们潜入吧！

### 作品和速记

让我们来看看制作如何[`VariableStatement`](https://tc39.es/ecma262/#prod-VariableStatement)已定义。乍一看，语法可能看起来有点可怕：

```grammar
VariableStatement[Yield, Await] :
  var VariableDeclarationList[+In, ?Yield, ?Await] ;
```

下标 （`[Yield, Await]`） 和前缀 （`+`在`+In`和`?`在`?Async`） mean？

该符号在本节中进行了解释[语法表示法](https://tc39.es/ecma262/#sec-grammar-notation).

下标是表示一组作品的简写，用于一组左侧符号，同时表达所有内容。左侧符号有两个参数，可扩展为四个“真实”左侧符号：`VariableStatement`,`VariableStatement_Yield`,`VariableStatement_Await`和`VariableStatement_Yield_Await`.

请注意，这里的平原`VariableStatement`意思是”`VariableStatement`没有`_Await`和`_Yield`".它不应该与混淆<code>变量声明<sub>\[屈服，等待]</sub></code>.

在制作的右侧，我们看到速记`+In`，意思是“使用版本`_In`“，以及`?Await`，意思是“使用版本`_Await`当且仅当左侧符号具有`_Await`“（与`?Yield`).

第三个速记，`~Foo`，意思是“使用没有的版本`_Foo`“，不用于此生产。

有了这些信息，我们可以像这样扩展制作：

```grammar
VariableStatement :
  var VariableDeclarationList_In ;

VariableStatement_Yield :
  var VariableDeclarationList_In_Yield ;

VariableStatement_Await :
  var VariableDeclarationList_In_Await ;

VariableStatement_Yield_Await :
  var VariableDeclarationList_In_Yield_Await ;
```

最终，我们需要找出两件事：

1.  它在哪里决定我们是否处于这种情况`_Await`或不带`_Await`?
2.  它在哪里有所作为 - 哪里的制作`Something_Await`和`Something`（无`_Await`） 分歧？

### `_Await`或否`_Await`?

让我们先解决问题 1。很容易猜到，非异步函数和异步函数在我们是否选择参数方面有所不同`_Await`是否为功能体。阅读异步函数声明的生产，我们发现[这](https://tc39.es/ecma262/#prod-AsyncFunctionBody):

```grammar
AsyncFunctionBody :
  FunctionBody[~Yield, +Await]
```

请注意，`AsyncFunctionBody`没有参数 — 它们被添加到`FunctionBody`在右侧。

如果我们扩大这种生产，我们得到：

```grammar
AsyncFunctionBody :
  FunctionBody_Await
```

换句话说，异步函数具有`FunctionBody_Await`，表示函数体，其中`await`被视为关键字。

另一方面，如果我们在非异步函数中，[相关生产](https://tc39.es/ecma262/#prod-FunctionDeclaration)是：

```grammar
FunctionDeclaration[Yield, Await, Default] :
  function BindingIdentifier[?Yield, ?Await] ( FormalParameters[~Yield, ~Await] ) { FunctionBody[~Yield, ~Await] }
```

(`FunctionDeclaration`有另一个生产，但它与我们的代码示例无关。

为了避免组合膨胀，让我们忽略`Default`在此特定生产中未使用的参数。

生产的扩展形式是：

```grammar
FunctionDeclaration :
  function BindingIdentifier ( FormalParameters ) { FunctionBody }

FunctionDeclaration_Yield :
  function BindingIdentifier_Yield ( FormalParameters ) { FunctionBody }

FunctionDeclaration_Await :
  function BindingIdentifier_Await ( FormalParameters ) { FunctionBody }

FunctionDeclaration_Yield_Await :
  function BindingIdentifier_Yield_Await ( FormalParameters ) { FunctionBody }
```

在这个生产中，我们总是得到`FunctionBody`和`FormalParameters`（无`_Yield`和没有`_Await`），因为它们被参数化为`[~Yield, ~Await]`在非扩大生产中。

函数名称的处理方式不同：它获取参数`_Await`和`_Yield`如果左侧符号有它们。

总结一下：异步函数有一个`FunctionBody_Await`和非异步函数具有`FunctionBody`（无`_Await`).由于我们谈论的是非生成器函数，因此我们的异步示例函数和非异步示例函数都是在没有参数化的情况下`_Yield`.

也许很难记住哪一个是`FunctionBody`以及哪个`FunctionBody_Await`.是`FunctionBody_Await`对于以下函数：`await`是一个标识符，或者对于函数，其中`await`是关键字吗？

您可以想到`_Await`参数含义”`await`是一个关键词”。这种方法也是面向未来的。想象一个新的关键字，`blob`正在添加，但仅在“blobby”函数内部添加。非 blobby 非异步非生成器仍然具有`FunctionBody`（无`_Await`,`_Yield`或`_Blob`），就像他们现在一样。Blobby 函数将有一个`FunctionBody_Blob`，异步 blobby 函数将具有`FunctionBody_Await_Blob`等等。我们仍然需要添加`Blob`下标到作品，但扩展的形式`FunctionBody`对于已经存在的功能保持不变。

### 禁止`await`作为标识符

接下来，我们需要找出如何`await`不允许作为标识符，如果我们在`FunctionBody_Await`.

我们可以进一步跟踪制作，看看`_Await`参数从`FunctionBody`一直到`VariableStatement`我们之前正在关注的生产。

因此，在异步函数中，我们将有一个`VariableStatement_Await`在非异步函数中，我们将有一个`VariableStatement`.

我们可以进一步跟踪生产并跟踪参数。我们已经看到了[`VariableStatement`](https://tc39.es/ecma262/#prod-VariableStatement):

```grammar
VariableStatement[Yield, Await] :
  var VariableDeclarationList[+In, ?Yield, ?Await] ;
```

所有作品[`VariableDeclarationList`](https://tc39.es/ecma262/#prod-VariableDeclarationList)只需按原样携带参数：

```grammar
VariableDeclarationList[In, Yield, Await] :
  VariableDeclaration[?In, ?Yield, ?Await]
```

（这里我们只显示[生产](https://tc39.es/ecma262/#prod-VariableDeclaration)与我们的示例相关。

```grammar
VariableDeclaration[In, Yield, Await] :
  BindingIdentifier[?Yield, ?Await] Initializer[?In, ?Yield, ?Await] opt
```

这`opt`速记意味着右侧符号是可选的;实际上有两个作品，一个有可选符号，一个没有。

在与我们的示例相关的简单案例中，`VariableStatement`由关键字组成`var`，后跟一个`BindingIdentifier`没有初始值设定项，并以分号结尾。

禁止或允许`await`奥斯纳`BindingIdentifier`，我们希望得到这样的东西：

```grammar
BindingIdentifier_Await :
  Identifier
  yield

BindingIdentifier :
  Identifier
  yield
  await
```

这将不允许`await`作为异步函数中的标识符，并允许它作为非异步函数内的标识符。

但是规范并没有像这样定义它，相反，我们发现这个[生产](https://tc39.es/ecma262/#prod-BindingIdentifier):

```grammar
BindingIdentifier[Yield, Await] :
  Identifier
  yield
  await
```

扩展后，这意味着以下作品：

```grammar
BindingIdentifier_Await :
  Identifier
  yield
  await

BindingIdentifier :
  Identifier
  yield
  await
```

（我们省略了`BindingIdentifier_Yield`和`BindingIdentifier_Yield_Await`在我们的示例中不需要。

这看起来像`await`和`yield`将始终允许作为标识符。这是怎么回事？整个博客文章是徒劳的吗？

### 静态语义来救援

事实证明，**静态语义**需要禁止`await`作为异步函数中的标识符。

静态语义描述静态规则，即在程序运行之前检查的规则。

在这种情况下，[静态语义`BindingIdentifier`](https://tc39.es/ecma262/#sec-identifiers-static-semantics-early-errors)定义以下语法导向规则：

> ```grammar
> BindingIdentifier[Yield, Await] : await
> ```
>
> 如果此生产具有<code><sub>\[等待]</sub></code>参数。

实际上，这禁止了`BindingIdentifier_Await : await`生产。

该规范解释说，之所以有这个生产，但通过静态语义将其定义为语法错误，是因为对自动分号插入（ASI）的干扰。

请记住，当我们无法根据语法生产解析一行代码时，ASI就会启动。ASI 尝试添加分号以满足语句和声明必须以分号结尾的要求。（我们将在后面的一集中更详细地描述ASI。

请考虑以下代码（规范中的示例）：

```js
async function too_few_semicolons() {
  let
  await 0;
}
```

如果语法不允许`await`作为标识符，ASI将启动并将代码转换为以下语法正确的代码，该代码还使用`let`作为标识符：

```js
async function too_few_semicolons() {
  let;
  await 0;
}
```

这种对ASI的干扰被认为太混乱了，因此使用静态语义来禁止`await`作为标识符。

### 禁止`StringValues`标识符数量

还有另一个相关规则：

> ```grammar
> BindingIdentifier : Identifier
> ```
>
> 如果此生产具有<code><sub>\[等待]</sub></code>参数和`StringValue`之`Identifier`是`"await"`.

起初这可能会令人困惑。[`Identifier`](https://tc39.es/ecma262/#prod-Identifier)定义如下：

<!-- markdownlint-disable no-inline-html -->

```grammar
Identifier :
  IdentifierName but not ReservedWord
```

<!-- markdownlint-enable no-inline-html -->

`await`是一个`ReservedWord`，那么如何才能`Identifier`永远`await`?

事实证明，`Identifier`不能`await`，但它可以是其他东西`StringValue`是`"await"`— 字符序列的不同表示形式`await`.

[标识符名称的静态语义](https://tc39.es/ecma262/#sec-identifier-names-static-semantics-stringvalue)定义如何`StringValue`的标识符名称被计算。例如，Unicode 转义序列`a`是`\u0061`所以`\u0061wait`具有`StringValue` `"await"`.`\u0061wait`不会被词法语法识别为关键字，而是`Identifier`.禁止在异步函数中将其用作变量名称的静态语义。

所以这有效：

```js
function old() {
  var \u0061wait;
}
```

这不会：

```js
async function modern() {
  var \u0061wait; // Syntax error
}
```

## 总结

在本集中，我们熟悉了词汇语法、句法语法以及用于定义句法语法的速记。例如，我们研究了禁止使用`await`作为异步函数内部的标识符，但允许它在非异步函数中。

句法语法的其他有趣部分，如自动分号插入和封面语法，将在后面的一集中介绍。敬请期待！
