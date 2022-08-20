***

标题： “了解 ECMAScript 规范，第 2 部分”
作者： '[Marja Hölttä](https://twitter.com/marjakh)，投机规格观众
化身：

*   马利亚-霍尔塔
    日期： 2020-03-02
    标签：
*   ECMAScript
*   了解 ECMAScript
    描述： '关于阅读 ECMAScript 规范的教程，第 2 部分'
    推文：“1234550773629014016”

***

让我们再练习一下我们真棒规格阅读技巧。如果你还没有看过上一集，现在是这样做的好时机！

[所有剧集](/blog/tags/understanding-ecmascript)

## 准备好第 2 部分了吗？

了解规范的一种有趣方法是从我们知道存在的JavaScript功能开始，并找出它是如何指定的。

> 警告！本集包含复制粘贴的算法[ECMAScript 规范](https://tc39.es/ecma262/)截至2020年2月。它们最终会过时。

我们知道属性是在原型链中查找的：如果一个对象没有我们尝试读取的属性，我们沿着原型链向上走，直到找到它（或者找到一个不再有原型的对象）。

例如：

```js
const o1 = { foo: 99 };
const o2 = {};
Object.setPrototypeOf(o2, o1);
o2.foo;
// → 99
```

## 原型行走在哪里定义？ { #prototype步行 }

让我们尝试找出此行为的定义位置。一个好的起点是列表[对象内部方法](https://tc39.es/ecma262/#sec-object-internal-methods-and-internal-slots).

两者兼而有之`[[GetOwnProperty]]`和`[[Get]]`— 我们对不限于以下版本的版本感兴趣*有*属性，因此我们将使用`[[Get]]`.

不幸的是，[属性描述符规范类型](https://tc39.es/ecma262/#sec-property-descriptor-specification-type)还有一个字段称为`[[Get]]`，因此在浏览规范时`[[Get]]`，我们需要仔细区分这两个独立的用法。

`[[Get]]`是一个**必要的内部方法**.**普通对象**实现基本内部方法的默认行为。**异国情调的物体**可以定义自己的内部方法`[[Get]]`这偏离了默认行为。在这篇文章中，我们专注于普通对象。

的默认实现`[[Get]]`委托给`OrdinaryGet`:

：：：ecmascript-algorithm

> **[`[[Get]] ( P, Receiver )`](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots-get-p-receiver)**
>
> 当`[[Get]]`内部方法`O`使用属性键调用`P`和 ECMAScript 语言值`Receiver`，则执行以下步骤：
>
> 1.  返回`? OrdinaryGet(O, P, Receiver)`.

我们很快就会看到`Receiver`是用作**此值**调用访问器属性的 getter 函数时。

`OrdinaryGet`定义如下：

：：：ecmascript-algorithm

> **[`OrdinaryGet ( O, P, Receiver )`](https://tc39.es/ecma262/#sec-ordinaryget)**
>
> 当抽象运算`OrdinaryGet`使用对象调用`O`、属性键`P`和 ECMAScript 语言值`Receiver`，则执行以下步骤：
>
> 1.  断言：`IsPropertyKey(P)`是`true`.
> 2.  让`desc`是`? O.[[GetOwnProperty]](P)`.
> 3.  如果`desc`是`undefined`然后
>     1.  让`parent`是`? O.[[GetPrototypeOf]]()`.
>     2.  如果`parent`是`null`返回`undefined`.
>     3.  返回`? parent.[[Get]](P, Receiver)`.
> 4.  如果`IsDataDescriptor(desc)`是`true`返回`desc.[[Value]]`.
> 5.  断言：`IsAccessorDescriptor(desc)`是`true`.
> 6.  让`getter`是`desc.[[Get]]`.
> 7.  如果`getter`是`undefined`返回`undefined`.
> 8.  返回`? Call(getter, Receiver)`.

原型链步道位于步骤 3 中：如果我们没有找到该属性作为自己的属性，则将原型称为`[[Get]]`委托给的方法`OrdinaryGet`再。如果我们仍然没有找到该属性，我们称之为原型的`[[Get]]`方法，委托给`OrdinaryGet`再次，依此类推，直到我们找到属性或到达没有原型的对象。

让我们看一下当我们访问时此算法是如何工作的`o2.foo`.首先，我们调用`OrdinaryGet`跟`O`存在`o2`和`P`存在`"foo"`.`O.[[GetOwnProperty]]("foo")`返回`undefined`因为`o2`没有自己的属性，称为`"foo"`，因此我们在步骤 3 中采用 if 分支。在步骤 3.a 中，我们设置`parent`到原型`o2`这是`o1`.`parent`莫`null`，因此我们不会在步骤 3.b 中返回。在步骤 3.c 中，我们调用父级的`[[Get]]`具有属性键的方法`"foo"`，然后返回它返回的任何内容。

父项 （`o1`） 是一个普通的对象，所以它的`[[Get]]`方法调用`OrdinaryGet`再次，这次与`O`存在`o1`和`P`存在`"foo"`.`o1`有自己的属性，称为`"foo"`，因此在步骤 2 中，`O.[[GetOwnProperty]]("foo")`返回关联的属性描述符，并将其存储在`desc`.

[属性描述符](https://tc39.es/ecma262/#sec-property-descriptor-specification-type)是一种规范类型。数据属性描述符将属性的值直接存储在`[[Value]]`田。访问器属性描述符将访问器函数存储在字段中`[[Get]]`和/或`[[Set]]`.在本例中，属性描述符与`"foo"`是数据属性描述符。

我们存储的数据属性描述符`desc`在步骤 2 中不是`undefined`，所以我们不采取`if`分支在步骤 3 中。接下来，我们执行步骤 4。属性描述符是一个数据属性描述符，因此我们返回其`[[Value]]`田`99`，在步骤 4 中，我们完成了。

## 什么是`Receiver`它来自哪里？{ #receiver }

这`Receiver`参数仅在步骤 8 中的访问器属性的情况下使用。它已作为**此值**调用访问器属性的 getter 函数时。

`OrdinaryGet`传递原件`Receiver`在整个递归过程中，保持不变（步骤3.c）。让我们找出`Receiver`原本是来的！

搜索以下地点`[[Get]]`叫做我们找到一个抽象的运算`GetValue`它对引用进行操作。引用是一种规范类型，由基值、引用的名称和严格的引用标志组成。在以下情况下`o2.foo`，则基值为对象`o2`，引用的名称是字符串`"foo"`，并且严格引用标志为`false`，因为示例代码很草率。

### 题外话：为什么参考文献不是记录？

旁听：参考不是记录，即使它听起来像是。它包含三个组件，这些组件同样可以表示为三个命名字段。参考不是仅仅因为历史原因的记录。

### 返回`GetValue`

让我们来看看如何`GetValue`定义：

：：：ecmascript-algorithm

> **[`GetValue ( V )`](https://tc39.es/ecma262/#sec-getvalue)**
>
> 1.  `ReturnIfAbrupt(V)`.
> 2.  如果`Type(V)`莫`Reference`返回`V`.
> 3.  让`base`是`GetBase(V)`.
> 4.  如果`IsUnresolvableReference(V)`是`true`，抛出一个`ReferenceError`例外。
> 5.  如果`IsPropertyReference(V)`是`true`然后
>     1.  如果`HasPrimitiveBase(V)`是`true`然后
>         1.  断言：在这种情况下，`base`永远不会`undefined`或`null`.
>         2.  设置`base`自`! ToObject(base)`.
>     2.  返回`? base.[[Get]](GetReferencedName(V), GetThisValue(V))`.
> 6.  还
>     1.  断言：`base`是环境记录。
>     2.  返回`? base.GetBindingValue(GetReferencedName(V), IsStrictReference(V))`

我们示例中的参考是`o2.foo`，这是一个属性引用。因此，我们采用分支5。我们不会在 5.a 中采用分支，因为基数 （`o2`） 不是[基元值](/blog/react-cliff#javascript-types)（数字、字符串、符号、BigInt、布尔值、未定义或空值）。

然后我们打电话给`[[Get]]`在步骤 5.b.这`Receiver`我们通过的是`GetThisValue(V)`.在本例中，它只是引用的基值：

：：：ecmascript-algorithm

> **[`GetThisValue( V )`](https://tc39.es/ecma262/#sec-getthisvalue)**
>
> 1.  断言：`IsPropertyReference(V)`是`true`.
> 2.  如果`IsSuperReference(V)`是`true`然后
>     1.  返回`thisValue`引用的组件`V`.
> 3.  返回`GetBase(V)`.

为`o2.foo`，我们不会在步骤 2 中采用分支，因为它不是超级引用（例如`super.foo`），但我们采取步骤 3 并返回引用的基值，即`o2`.

将所有内容拼凑在一起，我们发现我们设置了`Receiver`作为原始参考的基础，然后在原型链行走期间保持其不变。最后，如果我们找到的属性是访问器属性，我们使用`Receiver`作为**此值**呼叫它时。

特别是，**此值**在 getter 内部是指我们试图从中获取属性的原始对象，而不是我们在原型链行走期间找到该属性的对象。

让我们来试试吧！

```js
const o1 = { x: 10, get foo() { return this.x; } };
const o2 = { x: 50 };
Object.setPrototypeOf(o2, o1);
o2.foo;
// → 50
```

在此示例中，我们有一个名为`foo`我们为其定义了一个获取器。获取者返回`this.x`.

然后我们访问`o2.foo`- 获取者返回什么？

我们发现，当我们调用 getter 时，**此值**是我们最初尝试从中获取属性的对象，而不是我们找到它的对象。在这种情况下，**此值**是`o2`不`o1`.我们可以通过检查 getter 是否返回来验证这一点`o2.x`或`o1.x`，事实上，它会返回`o2.x`.

它的工作原理！我们能够根据我们在规范中阅读的内容来预测此代码片段的行为。

## 访问属性 — 为什么它调用`[[Get]]`?{ #property-access-get }

规范在哪里说对象内部方法`[[Get]]`将在访问像这样的属性时被调用`o2.foo`?当然，这必须在某个地方定义。不要相信我的话！

我们发现 Object 内部方法`[[Get]]`从抽象操作调用`GetValue`它对引用进行操作。但是在哪里`GetValue`从？

### 运行时语义`MemberExpression`{ #memberexpression }

规范的语法规则定义了语言的语法。[运行时语义](https://tc39.es/ecma262/#sec-runtime-semantics)定义语法构造的“含义”（如何在运行时评估它们）。

如果您不熟悉[上下文无关语法](https://en.wikipedia.org/wiki/Context-free_grammar)，现在看看是个好主意！

我们将在后面的一集中更深入地研究语法规则，让我们现在保持简单！特别是，我们可以忽略下标（`Yield`,`Await`等等）在本集的制作中。

以下作品描述了什么[`MemberExpression`](https://tc39.es/ecma262/#prod-MemberExpression)看来：

```grammar
MemberExpression :
  PrimaryExpression
  MemberExpression [ Expression ]
  MemberExpression . IdentifierName
  MemberExpression TemplateLiteral
  SuperProperty
  MetaProperty
  new MemberExpression Arguments
```

在这里，我们有7个作品`MemberExpression`.一个`MemberExpression`可以只是一个`PrimaryExpression`.或者，一个`MemberExpression`可以从另一个构造`MemberExpression`和`Expression`通过将它们拼凑在一起：`MemberExpression [ Expression ]`例如`o2['foo']`.或者它可以是`MemberExpression . IdentifierName`例如`o2.foo`— 这是与我们示例相关的生产。

生产的运行时语义`MemberExpression : MemberExpression . IdentifierName`定义评估时要执行的步骤集：

：：：ecmascript-algorithm

> **[运行时语义：评估`MemberExpression : MemberExpression . IdentifierName`](https://tc39.es/ecma262/#sec-property-accessors-runtime-semantics-evaluation)**
>
> 1.  让`baseReference`是评估的结果`MemberExpression`.
> 2.  让`baseValue`是`? GetValue(baseReference)`.
> 3.  如果代码与此匹配`MemberExpression`是严格模式代码，让`strict`是`true`;否则让`strict`是`false`.
> 4.  返回`? EvaluatePropertyAccessWithIdentifierKey(baseValue, IdentifierName, strict)`.

算法委托给抽象操作`EvaluatePropertyAccessWithIdentifierKey`，所以我们也需要阅读它：

：：：ecmascript-algorithm

> **[`EvaluatePropertyAccessWithIdentifierKey( baseValue, identifierName, strict )`](https://tc39.es/ecma262/#sec-evaluate-property-access-with-identifier-key)**
>
> 抽象运算`EvaluatePropertyAccessWithIdentifierKey`将值作为参数`baseValue`，解析节点`identifierName`和布尔参数`strict`.它执行以下步骤：
>
> 1.  断言：`identifierName`是一个`IdentifierName`
> 2.  让`bv`是`? RequireObjectCoercible(baseValue)`.
> 3.  让`propertyNameString`是`StringValue`之`identifierName`.
> 4.  返回引用类型的值，其基值组件为`bv`，其引用的名称组件为`propertyNameString`，其严格引用标志为`strict`.

那是：`EvaluatePropertyAccessWithIdentifierKey`构造一个引用，该引用使用所提供的`baseValue`作为基数，字符串值`identifierName`作为属性名称，以及`strict`作为严格模式标志。

最终，此引用将传递给`GetValue`.这在规范中的多个位置进行了定义，具体取决于引用最终的使用方式。

### `MemberExpression`作为参数

在我们的示例中，我们使用属性访问作为参数：

```js
console.log(o2.foo);
```

在这种情况下，行为在 运行时语义中定义`ArgumentList`生产调用`GetValue`在参数上：

：：：ecmascript-algorithm

> **[运行时语义：`ArgumentListEvaluation`](https://tc39.es/ecma262/#sec-argument-lists-runtime-semantics-argumentlistevaluation)**
>
> `ArgumentList : AssignmentExpression`
>
> 1.  让`ref`是评估的结果`AssignmentExpression`.
> 2.  让`arg`是`? GetValue(ref)`.
> 3.  返回其唯一项目为`arg`.

`o2.foo`看起来不像`AssignmentExpression`但它是一个，所以这个生产是适用的。要找出原因，您可以查看此内容[额外内容](/blog/extras/understanding-ecmascript-part-2-extra)，但此时并非绝对必要。

这`AssignmentExpression`在步骤 1 中是`o2.foo`.`ref`，则评估结果`o2.foo`，是上面提到的参考。在步骤 2 中，我们调用`GetValue`在上面。因此，我们知道对象内部方法`[[Get]]`将被调用，并且将发生原型链行走。

## 总结

在本集中，我们研究了规范如何定义语言特征，在本例中为原型查找，跨越所有不同的层：触发特征的语法构造和定义该特征的算法。
