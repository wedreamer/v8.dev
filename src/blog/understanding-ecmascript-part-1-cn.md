***

标题： “了解 ECMAScript 规范，第 1 部分”
作者： '[Marja Hölttä](https://twitter.com/marjakh)，投机规格观众
化身：

*   马利亚-霍尔塔
    日期： 2020-02-03 13：33：37
    标签：
*   ECMAScript
*   了解 ECMAScript
    描述： '关于阅读 ECMAScript 规范的教程'
    推文：“1224363301146189824”

***

[所有剧集](/blog/tags/understanding-ecmascript)

在本文中，我们在规范中采用一个简单的函数，并尝试理解表示法。我们走吧！

## 前言

即使你知道JavaScript，阅读它的语言规范，[ECMAScript 语言规范，或简称 ECMAScript 规范](https://tc39.es/ecma262/)，可能非常令人生畏。至少这是我第一次开始阅读它时的感受。

让我们从一个具体的例子开始，并遍历规范来理解它。下面的代码演示了`Object.prototype.hasOwnProperty`:

```js
const o = { foo: 1 };
o.hasOwnProperty('foo'); // true
o.hasOwnProperty('bar'); // false
```

在示例中，`o`没有名为`hasOwnProperty`，所以我们走上原型链并寻找它。我们在`o`的原型，即`Object.prototype`.

描述如何`Object.prototype.hasOwnProperty`works，规范使用类似伪代码的描述：

：：：ecmascript-algorithm

> **[`Object.prototype.hasOwnProperty(V)`](https://tc39.es/ecma262#sec-object.prototype.hasownproperty)**
>
> 当`hasOwnProperty`方法使用参数调用`V`，则执行以下步骤：
>
> 1.  让`P`是`? ToPropertyKey(V)`.
> 2.  让`O`是`? ToObject(this value)`.
> 3.  返回`? HasOwnProperty(O, P)`.
>     :::

...和。。。

：：：ecmascript-algorithm

> **[`HasOwnProperty(O, P)`](https://tc39.es/ecma262#sec-hasownproperty)**
>
> 抽象运算`HasOwnProperty`用于确定对象是否具有具有指定属性键的自己的属性。返回一个布尔值。使用参数调用操作`O`和`P`哪里`O`是对象和`P`是属性键。此抽象操作执行以下步骤：
>
> 1.  断言：`Type(O)`是`Object`.
> 2.  断言：`IsPropertyKey(P)`是`true`.
> 3.  让`desc`是`? O.[[GetOwnProperty]](P)`.
> 4.  如果`desc`是`undefined`返回`false`.
> 5.  返回`true`.
>     :::

但什么是“抽象操作”呢？里面有什么东西`[[ ]]`?为什么有一个`?`在函数前面？断言是什么意思？

让我们来一探究竟！

## 语言类型和规范类型

让我们从看起来很熟悉的东西开始。规范使用的值，例如`undefined`,`true`和`false`，我们已经从 JavaScript 中知道了。他们都是[**语言值**](https://tc39.es/ecma262/#sec-ecmascript-language-types)，值**语言类型**规范也定义了这一点。

规范还在内部使用语言值，例如，内部数据类型可能包含一个字段，其可能值为`true`和`false`.相比之下，JavaScript引擎通常不会在内部使用语言值。例如，如果 JavaScript 引擎是用 C++ 编写的，它通常使用C++`true`和`false`（而不是它对JavaScript的内部表示`true`和`false`).

除语言类型外，该规范还使用[**规格类型**](https://tc39.es/ecma262/#sec-ecmascript-specification-types)，这些类型仅在规范中出现，但不在 JavaScript 语言中出现。JavaScript引擎不需要（但可以自由地）实现它们。在这篇博客文章中，我们将了解规范类型记录（及其子类型完成记录）。

## 抽象运算

[**抽象运算**](https://tc39.es/ecma262/#sec-abstract-operations)是 ECMAScript 规范中定义的函数;它们是为了简明扼要地编写规范而定义的。JavaScript引擎不必将它们作为引擎内部的单独函数实现。它们不能直接从 JavaScript 调用。

## 内部插槽和内部方法

[**内部插槽**和**内部方法**](https://tc39.es/ecma262/#sec-object-internal-methods-and-internal-slots)使用括在`[[ ]]`.

内部槽是 JavaScript 对象或规范类型的数据成员。它们用于存储对象的状态。内部方法是 JavaScript 对象的成员函数。

例如，每个 JavaScript 对象都有一个内部槽`[[Prototype]]`和内部方法`[[GetOwnProperty]]`.

内部插槽和方法无法从 JavaScript 访问。例如，您不能访问`o.[[Prototype]]`或致电`o.[[GetOwnProperty]]()`.JavaScript引擎可以实现它们供自己的内部使用，但不必这样做。

有时，内部方法委托给名称相似的抽象操作，例如在普通对象的情况下`[[GetOwnProperty]]:`

：：：ecmascript-algorithm

> **[`[[GetOwnProperty]](P)`](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots-getownproperty-p)**
>
> 当`[[GetOwnProperty]]`内部方法`O`使用属性键调用`P`，则执行以下步骤：
>
> 1.  返回`! OrdinaryGetOwnProperty(O, P)`.
>     :::

（我们将在下一章中了解感叹号的含义。

`OrdinaryGetOwnProperty`不是内部方法，因为它不与任何对象相关联;相反，它所操作的对象作为参数传递。

`OrdinaryGetOwnProperty`被称为“普通”，因为它在普通对象上操作。ECMAScript 对象可以是**普通**或**异国情调**.普通对象必须具有一组方法的默认行为，称为**基本的内部方法**.如果某个对象偏离默认行为，则它是外来的。

最知名的奇异物体是`Array`，因为它的长度属性以非默认方式运行：设置`length`属性可以从`Array`.

基本的内部方法是列出的方法[这里](https://tc39.es/ecma262/#table-5).

## 完成记录

问号和感叹号呢？要了解它们，我们需要研究[**完成记录**](https://tc39.es/ecma262/#sec-completion-record-specification-type)!

完成记录是一种规范类型（仅为规范目的而定义）。JavaScript 引擎不必具有相应的内部数据类型。

完成记录是一种“记录”，即具有一组固定的命名字段的数据类型。完成记录有三个字段：

：：：表包装器
|名称|描述 |
|------------ |------------------------------------------------------------------------------------------------------------------------------------------ |
|`[[Type]]`|其中之一：`normal`,`break`,`continue`,`return`或`throw`.除`normal`是**突然完成**.                  |
|`[[Value]]`|完成时生成的值，例如，函数的返回值或异常（如果引发异常）。|
|`[[Target]]`|用于定向控制权转移（与此博客文章无关）。                                                                    |
:::

每个抽象操作都隐式返回一个完成记录。即使它看起来像一个抽象操作将返回一个简单的类型，如布尔值，它也隐式包装到一个完成记录中，类型为`normal`（请参见[隐式完成值](https://tc39.es/ecma262/#sec-implicit-completion-values)).

注1：规范在这方面并不完全一致;有一些帮助器函数返回裸值，其返回值按原样使用，而不从完成记录中提取值。这通常从上下文中可以清楚地看出。

注 2：规范编辑器正在考虑使“完成记录”处理更加明确。

如果算法引发异常，则意味着返回完成记录`[[Type]]` `throw`谁的`[[Value]]`是异常对象。我们将忽略`break`,`continue`和`return`类型现在。

[`ReturnIfAbrupt(argument)`](https://tc39.es/ecma262/#sec-returnifabrupt)意味着采取以下步骤：

：：：ecmascript-algorithm

<!-- markdownlint-disable blanks-around-lists -->

> 1.  如果`argument`是突兀的，回来的`argument`
> 2.  设置`argument`自`argument.[[Value]]`.

<!-- markdownlint-enable blanks-around-lists -->

:::

也就是说，我们检查完成记录;如果是突然完成，我们会立即返回。否则，我们从完成记录中提取值。

`ReturnIfAbrupt`可能看起来像一个函数调用，但事实并非如此。它导致以下功能`ReturnIfAbrupt()`发生返回，而不是`ReturnIfAbrupt`函数本身。它的行为更像是类C语言中的宏。

`ReturnIfAbrupt`可以这样使用：

：：：ecmascript-algorithm

<!-- markdownlint-disable blanks-around-lists -->

> 1.  让`obj`是`Foo()`.(`obj`是完成记录。
> 2.  `ReturnIfAbrupt(obj)`.
> 3.  `Bar(obj)`.（如果我们还在这里，`obj`是从“完成记录”中提取的值。

<!-- markdownlint-enable blanks-around-lists -->

:::

现在[问号](https://tc39.es/ecma262/#sec-returnifabrupt-shorthands)发挥作用：`? Foo()`等效于`ReturnIfAbrupt(Foo())`.使用速记是实用的：我们不需要每次都显式编写错误处理代码。

同样地`Let val be ! Foo()`等效于：

：：：ecmascript-algorithm

<!-- markdownlint-disable blanks-around-lists -->

> 1.  让`val`是`Foo()`.
> 2.  断言：`val`不是突然完成。
> 3.  设置`val`自`val.[[Value]]`.

<!-- markdownlint-enable blanks-around-lists -->

:::

利用这些知识，我们可以重写`Object.prototype.hasOwnProperty`喜欢这个：

：：：ecmascript-algorithm

> **`Object.prototype.hasOwnProperty(V)`**
>
> 1.  让`P`是`ToPropertyKey(V)`.
> 2.  如果`P`是突然完成，返回`P`
> 3.  设置`P`自`P.[[Value]]`
> 4.  让`O`是`ToObject(this value)`.
> 5.  如果`O`是突然完成，返回`O`
> 6.  设置`O`自`O.[[Value]]`
> 7.  让`temp`是`HasOwnProperty(O, P)`.
> 8.  如果`temp`是突然完成，返回`temp`
> 9.  让`temp`是`temp.[[Value]]`
> 10. 返回`NormalCompletion(temp)`
>     :::

...我们可以重写`HasOwnProperty`喜欢这个：

：：：ecmascript-algorithm

> **`HasOwnProperty(O, P)`**
>
> 1.  断言：`Type(O)`是`Object`.
> 2.  断言：`IsPropertyKey(P)`是`true`.
> 3.  让`desc`是`O.[[GetOwnProperty]](P)`.
> 4.  如果`desc`是突然完成，返回`desc`
> 5.  设置`desc`自`desc.[[Value]]`
> 6.  如果`desc`是`undefined`返回`NormalCompletion(false)`.
> 7.  返回`NormalCompletion(true)`.
>     :::

我们还可以重写`[[GetOwnProperty]]`不带感叹号的内部方法：

：：：ecmascript-algorithm

<!-- markdownlint-disable blanks-around-lists -->

> **`O.[[GetOwnProperty]]`**
>
> 1.  让`temp`是`OrdinaryGetOwnProperty(O, P)`.
> 2.  断言：`temp`不是突然完成。
> 3.  让`temp`是`temp.[[Value]]`.
> 4.  返回`NormalCompletion(temp)`.

<!-- markdownlint-enable blanks-around-lists -->

:::

在这里，我们假设`temp`是一个全新的临时变量，不会与其他任何东西发生冲突。

我们还使用了这样一个知识，即当 return 语句返回的不是完成记录时，它被隐式包装在`NormalCompletion`.

### 侧轨：`Return ? Foo()`

规范使用符号`Return ? Foo()`— 为什么是问号？

`Return ? Foo()`扩展到：

：：：ecmascript-algorithm

<!-- markdownlint-disable blanks-around-lists -->

> 1.  让`temp`是`Foo()`.
> 2.  如果`temp`是突然完成，返回`temp`.
> 3.  设置`temp`自`temp.[[Value]]`.
> 4.  返回`NormalCompletion(temp)`.

<!-- markdownlint-enable blanks-around-lists -->

:::

这与`Return Foo()`;对于突然和正常完成，它的行为方式相同。

`Return ? Foo()`仅用于编辑原因，以使其更明确`Foo`返回完成记录。

## 断言

在规范中断言算法的不变条件。为了清楚起见，添加了它们，但不会向实现添加任何要求 - 实现不需要检查它们。

## 继续前进

抽象操作委托给其他抽象操作（见下图），但基于这篇博客文章，我们应该能够弄清楚它们的作用。我们将遇到属性描述符，这只是另一种规范类型。

![Function call graph starting from Object.prototype.hasOwnProperty](/\_img/understanding-ecmascript-part-1/call-graph.svg)

## 总结

我们通读一个简单的方法——`Object.prototype.hasOwnProperty`— 和**抽象操作**它调用。我们熟悉了速记`?`和`!`与错误处理相关。我们遇到**语言类型**,**规格类型**,**内部插槽**和**内部方法**.

## 有用的链接

[如何阅读 ECMAScript 规范](https://timothygu.me/es-howto/)：一个教程，涵盖了这篇文章中涵盖的大部分材料，从一个稍微不同的角度。
