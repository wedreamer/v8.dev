***

标题：“逻辑分配”
作者： '郭淑宇 （[@\_shu](https://twitter.com/\_shu))'
化身：

*   “舒玉国”
    日期： 2020-05-07
    标签：
*   ECMAScript
*   ES2021
*   节点.js 16
    描述： 'JavaScript 现在支持具有逻辑运算的复合赋值。
    推文：“1258387483823345665”

***

JavaScript 支持一系列[复合赋值运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Assignment_Operators)让程序员简洁地表达二进制操作和赋值。目前，仅支持数学运算或按位运算。

缺少的是将逻辑操作与分配相结合的能力。直到现在！JavaScript 现在支持使用新运算符进行逻辑赋值`&&=`,`||=`和`??=`.

## 逻辑赋值运算符

在深入研究新运算符之前，让我们对现有的复合赋值运算符进行复习。例如，含义`lhs += rhs`大致相当于`lhs = lhs + rhs`.这种粗略等价适用于所有现有运算符`@=`哪里`@`代表二元运算符，如`+`或`|`.值得注意的是，严格来说，这只有在以下情况下才是正确的：`lhs`是一个变量。对于表达式中更复杂的左侧，例如`obj[computedPropertyName()] += rhs`，左侧仅评估一次。

现在让我们深入了解新的运算符。与现有的运营商相比，`lhs @= rhs`不大致意味着`lhs = lhs @ rhs`什么时候`@`是一个逻辑运算：`&&`,`||`或`??`.

```js
// As an additional review, here is the semantics of logical and:
x && y
// → y when x is truthy
// → x when x is not truthy

// First, logical and assignment. The two lines following this
// comment block are equivalent.
// Note that like existing compound assignment operators, more complex
// left-hand sides are only evaluated once.
x &&= y;
x && (x = y);

// The semantics of logical or:
x || y
// → x when x is truthy
// → y when x is not truthy

// Similarly, logical or assignment:
x ||= y;
x || (x = y);

// The semantics of nullish coalescing operator:
x ?? y
// → y when x is nullish (null or undefined)
// → x when x is not nullish

// Finally, nullish coalescing assignment:
x ??= y;
x ?? (x = y);
```

## 短路语义

与数学和按位对应物不同，逻辑赋值遵循其各自逻辑运算的短路行为。他们*只*如果逻辑运算将评估右侧，则执行赋值。

起初，这似乎令人困惑。为什么不像其他复合赋值一样无条件地赋值到左侧呢？

这种差异有一个很好的实际原因。将逻辑运算与赋值相结合时，赋值可能会导致副作用，该副作用应根据该逻辑运算的结果有条件地发生。无条件地引起副作用会对程序的性能甚至正确性产生负面影响。

让我们通过一个函数的两个版本的示例来具体说明这一点，该函数在元素中设置默认消息。

```js
// Display a default message if it doesn’t override anything.
// Only assigns to innerHTML if it’s empty. Doesn’t cause inner
// elements of msgElement to lose focus.
function setDefaultMessage() {
  msgElement.innerHTML ||= '<p>No messages<p>';
}

// Display a default message if it doesn’t override anything.
// Buggy! May cause inner elements of msgElement to
// lose focus every time it’s called.
function setDefaultMessageBuggy() {
  msgElement.innerHTML = msgElement.innerHTML || '<p>No messages<p>';
}
```

：：：备注
**注意：**因为`innerHTML`属性是[指定](https://w3c.github.io/DOM-Parsing/#dom-innerhtml-innerhtml)返回空字符串而不是`null`或`undefined`,`||=`必须使用而不是`??=`.编写代码时，请记住，许多 Web API 不使用`null`或`undefined`表示空或缺席。
:::

在 HTML 中，分配给`.innerHTML`元素上的属性是破坏性的。删除内部子级，并插入从新分配的字符串中分析的新子级。即使新字符串与旧字符串相同，也会导致额外的工作和内部元素失去焦点。出于这种不引起不需要的副作用的实际原因，逻辑赋值运算符的语义使赋值短路。

以下列方式考虑与其他复合赋值运算符的对称性可能会有所帮助。数学运算符和按位运算符是无条件的，因此赋值也是无条件的。逻辑运算符是有条件的，因此赋值也是有条件的。

## 逻辑分配支持 { #support }

<feature-support chrome="85"
              firefox="79 https://bugzilla.mozilla.org/show_bug.cgi?id=1629106"
              safari="14 https://developer.apple.com/documentation/safari-release-notes/safari-14-beta-release-notes#New-Features:~:text=Added%20logical%20assignment%20operator%20support."
              nodejs="16"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-logical-assignment-operators"></feature-support>
