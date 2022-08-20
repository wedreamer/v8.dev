***

标题： “了解 ECMAScript 规范，第 4 部分”
作者： '[Marja Hölttä](https://twitter.com/marjakh)，投机规格观众
化身：

*   马利亚-霍尔塔
    日期： 2020-05-19
    标签：
*   ECMAScript
*   了解 ECMAScript
    描述： '关于阅读 ECMAScript 规范的教程'
    推文：“1262815621756014594”

***

[所有剧集](/blog/tags/understanding-ecmascript)

## 同时在网络的其他部分

[杰森·奥伦多夫](https://github.com/jorendorff)从 Mozilla 发布[对JS语法怪癖的深入分析](https://github.com/mozilla-spidermonkey/jsparagus/blob/master/js-quirks.md#readme).尽管实现细节不同，但每个JS引擎都面临着与这些怪癖相同的问题。

## 封面语法

在本集中，我们将更深入地研究*覆盖语法*.它们是一种为语法结构指定语法的方法，这些语法结构起初看起来模棱两可。

同样，我们将跳过下标`[In, Yield, Await]`为了简洁起见，因为它们对这篇博客文章并不重要。看[第 3 部分](/blog/understanding-ecmascript-part-3)以解释其含义和用法。

## 有限的展望

通常，解析器根据有限的前瞻（固定数量的后续令牌）决定使用哪个生产。

在某些情况下，下一个令牌确定要明确使用的生产。[例如](https://tc39.es/ecma262/#prod-UpdateExpression):

```grammar
UpdateExpression :
  LeftHandSideExpression
  LeftHandSideExpression ++
  LeftHandSideExpression --
  ++ UnaryExpression
  -- UnaryExpression
```

如果我们正在解析`UpdateExpression`下一个令牌是`++`或`--`，我们立即知道要使用的生产。如果下一个令牌都不是，它仍然不是太糟糕：我们可以解析一个`LeftHandSideExpression`从我们所处的位置开始，弄清楚在解析它后该怎么做。

如果令牌后面的`LeftHandSideExpression`是`++`，生产使用是`UpdateExpression : LeftHandSideExpression ++`.案例`--`是相似的。如果令牌后面`LeftHandSideExpression`都不是`++`也不`--`，我们使用生产`UpdateExpression : LeftHandSideExpression`.

### 箭头函数参数列表还是带括号的表达式？

将箭头函数参数列表与带括号的表达式区分开来更为复杂。

例如：

```js
let x = (a,
```

这是箭头函数的开始吗，就像这样？

```js
let x = (a, b) => { return a + b };
```

或者也许是一个带括号的表达式，就像这样？

```js
let x = (a, 3);
```

括号中的任何东西都可以任意长 - 我们无法根据有限数量的令牌知道它是什么。

让我们想象一下，我们有以下简单的制作：

```grammar
AssignmentExpression :
  ...
  ArrowFunction
  ParenthesizedExpression

ArrowFunction :
  ArrowParameterList => ConciseBody
```

现在，我们无法选择使用有限前瞻的生产。如果我们必须解析一个`AssignmentExpression`下一个令牌是`(`，我们将如何决定下一步解析什么？我们可以解析`ArrowParameterList`或`ParenthesizedExpression`，但是我们的猜测可能会出错。

### 非常宽容的新符号：`CPEAAPL`

规范通过引入符号来解决这个问题`CoverParenthesizedExpressionAndArrowParameterList`(`CPEAAPL`简称）。`CPEAAPL`是一个符号，实际上是一个`ParenthesizedExpression`或`ArrowParameterList`在幕后，但我们还不知道是哪一个。

这[制作](https://tc39.es/ecma262/#prod-CoverParenthesizedExpressionAndArrowParameterList)为`CPEAAPL`非常宽容，允许所有可能发生在`ParenthesizedExpression`s 和 in`ArrowParameterList`s:

```grammar
CPEAAPL :
  ( Expression )
  ( Expression , )
  ( )
  ( ... BindingIdentifier )
  ( ... BindingPattern )
  ( Expression , ... BindingIdentifier )
  ( Expression , ... BindingPattern )
```

例如，以下表达式是有效的`CPEAAPL`s:

```js
// Valid ParenthesizedExpression and ArrowParameterList:
(a, b)
(a, b = 1)

// Valid ParenthesizedExpression:
(1, 2, 3)
(function foo() { })

// Valid ArrowParameterList:
()
(a, b,)
(a, ...b)
(a = 1, ...b)

// Not valid either, but still a CPEAAPL:
(1, ...b)
(1, )
```

尾随逗号和`...`只能出现在`ArrowParameterList`.某些构造，如`b = 1`可以同时出现在两者中，但它们具有不同的含义：内部`ParenthesizedExpression`这是一个任务，里面`ArrowParameterList`它是具有默认值的参数。数字和其他`PrimaryExpressions`无效的参数名称（或参数解构模式）只能出现在`ParenthesizedExpression`.但它们都可能发生在`CPEAAPL`.

### 用`CPEAAPL`在作品中

现在我们可以使用非常宽松的`CPEAAPL`在[`AssignmentExpression`制作](https://tc39.es/ecma262/#prod-AssignmentExpression).（注：`ConditionalExpression`导致`PrimaryExpression`通过一个长长的生产链，这里没有显示。

```grammar
AssignmentExpression :
  ConditionalExpression
  ArrowFunction
  ...

ArrowFunction :
  ArrowParameters => ConciseBody

ArrowParameters :
  BindingIdentifier
  CPEAAPL

PrimaryExpression :
  ...
  CPEAAPL

```

想象一下，我们再次处于需要解析`AssignmentExpression`下一个令牌是`(`.现在我们可以解析一个`CPEAAPL`并在以后弄清楚要使用什么生产。我们是否解析`ArrowFunction`或`ConditionalExpression`，则要解析的下一个符号是`CPEAAPL`无论如何！

解析完`CPEAAPL`，我们可以决定使用哪个生产来制作原版`AssignmentExpression`（包含`CPEAAPL`).此决定基于以下令牌做出的`CPEAAPL`.

如果令牌是`=>`，我们使用生产：

```grammar
AssignmentExpression :
  ArrowFunction
```

如果令牌是其他东西，我们使用生产：

```grammar
AssignmentExpression :
  ConditionalExpression
```

例如：

```js
let x = (a, b) => { return a + b; };
//      ^^^^^^
//     CPEAAPL
//             ^^
//             The token following the CPEAAPL

let x = (a, 3);
//      ^^^^^^
//     CPEAAPL
//            ^
//            The token following the CPEAAPL
```

在这一点上，我们可以保持`CPEAAPL`，并继续解析程序的其余部分。例如，如果`CPEAAPL`在`ArrowFunction`，我们还不需要查看它是否是有效的箭头函数参数列表 - 这可以稍后完成。（现实世界的解析器可能会选择立即进行有效性检查，但从规范的角度来看，我们不需要这样做。

### 限制CPEAPL

正如我们之前所看到的，语法产生`CPEAAPL`非常宽容并允许构造（例如`(1, ...a)`），这些永远无效。根据语法解析程序后，我们需要禁止相应的非法构造。

规范通过添加以下限制来实现此目的：

：：：ecmascript-algorithm

> [静态语义：早期错误](https://tc39.es/ecma262/#sec-grouping-operator-static-semantics-early-errors)
>
> `PrimaryExpression : CPEAAPL`
>
> 如果出现以下情况，则为语法错误`CPEAAPL`未覆盖`ParenthesizedExpression`.

：：：ecmascript-algorithm

> [补充语法](https://tc39.es/ecma262/#sec-primary-expression)
>
> 处理生产实例时
>
> `PrimaryExpression : CPEAAPL`
>
> 的解释`CPEAAPL`使用以下语法进行优化：
>
> `ParenthesizedExpression : ( Expression )`

这意味着：如果`CPEAAPL`发生在`PrimaryExpression`在语法树中，它实际上是一个`ParenthesizedExpression`这是它唯一有效的生产。

`Expression`永远不能是空的，所以`( )`无效`ParenthesizedExpression`.逗号分隔列表，如`(1, 2, 3)`由创建者[逗号运算符](https://tc39.es/ecma262/#sec-comma-operator):

```grammar
Expression :
  AssignmentExpression
  Expression , AssignmentExpression
```

同样，如果`CPEAAPL`发生在`ArrowParameters`，则以下限制适用：

：：：ecmascript-algorithm

> [静态语义：早期错误](https://tc39.es/ecma262/#sec-arrow-function-definitions-static-semantics-early-errors)
>
> `ArrowParameters : CPEAAPL`
>
> 如果出现以下情况，则为语法错误`CPEAAPL`未覆盖`ArrowFormalParameters`.

：：：ecmascript-algorithm

> [补充语法](https://tc39.es/ecma262/#sec-arrow-function-definitions)
>
> 生产时
>
> `ArrowParameters`:`CPEAAPL`
>
> 是公认的以下语法用于细化解释`CPEAAPL`:
>
> `ArrowFormalParameters :`
> `( UniqueFormalParameters )`

### 其他封面语法

除了`CPEAAPL`，规范将覆盖语法用于其他看起来模棱两可的结构。

`ObjectLiteral`用作封面语法`ObjectAssignmentPattern`这发生在箭头函数参数列表内。这意味着`ObjectLiteral`允许不能在实际对象文本中出现的构造。

```grammar
ObjectLiteral :
  ...
  { PropertyDefinitionList }

PropertyDefinition :
  ...
  CoverInitializedName

CoverInitializedName :
  IdentifierReference Initializer

Initializer :
  = AssignmentExpression
```

例如：

```js
let o = { a = 1 }; // syntax error

// Arrow function with a destructuring parameter with a default
// value:
let f = ({ a = 1 }) => { return a; };
f({}); // returns 1
f({a : 6}); // returns 6
```

异步箭头函数在有限前瞻中也显得模棱两可：

```js
let x = async(a,
```

这是对名为`async`还是异步箭头函数？

```js
let x1 = async(a, b);
let x2 = async();
function async() { }

let x3 = async(a, b) => {};
let x4 = async();
```

为此，语法定义了一个封面语法符号`CoverCallExpressionAndAsyncArrowHead`其工作原理类似于`CPEAAPL`.

## 总结

在本集中，我们研究了规范如何定义覆盖语法，并在我们无法基于有限前瞻识别当前句法结构的情况下使用它们。

特别是，我们研究了将箭头函数参数列表与括号表达式区分开来，以及规范如何使用覆盖语法首先解析看起来不明确的结构，然后使用静态语义规则对其进行限制。
