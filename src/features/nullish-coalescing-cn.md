***

标题： “空合并”
作者： '贾斯汀·里奇韦尔'
化身：

*   “贾斯汀-里奇韦尔”
    日期： 2019-09-17
    标签：
*   ECMAScript
*   ES2020
    描述： “JavaScript 空合并运算符启用更安全的默认表达式。
    推文：“1173971116865523714”

***

这[空合并提案](https://github.com/tc39/proposal-nullish-coalescing/)(`??`） 添加了一个新的短路运算符，用于处理默认值。

您可能已经熟悉其他短路操作器`&&`和`||`.这两个运算符都处理“真实”和“虚假”值。想象一下代码示例`lhs && rhs`.如果`lhs`（阅读，*左侧*） 是 falsy，表达式的计算结果为`lhs`.否则，其计算结果为`rhs`（阅读，*右侧*).代码示例的情况正好相反`lhs || rhs`.如果`lhs`是真心的，表达式的计算结果为`lhs`.否则，其计算结果为`rhs`.

但是，“真理”和“虚假”究竟是什么意思呢？在规格术语中，它等同于[`ToBoolean`](https://tc39.es/ecma262/#sec-toboolean)抽象操作。对于我们这些普通的 JavaScript 开发者来说，**万事**是真实的，除了虚假的价值观`undefined`,`null`,`false`,`0`,`NaN`和空字符串`''`.（从技术上讲，与`document.all`也是虚假的，但我们稍后会谈到这一点。

那么，有什么问题吗？`&&`和`||`?为什么我们需要一个新的零合并运算符？这是因为这种对真伪的定义并不适合所有场景，这导致了错误。想象一下：

```js
function Component(props) {
  const enable = props.enabled || true;
  // …
}
```

在这个例子中，让我们处理`enabled`属性作为可选的布尔属性，用于控制是否启用组件中的某些功能。意思是，我们可以显式设置`enabled``true`或`false`.但是，因为它是一个*自选*属性，我们可以隐式将其设置为`undefined`通过根本不设置它。如果是`undefined`我们希望将其视为组件`enabled = true`（其默认值）。

到目前为止，您可能可以通过代码示例发现该错误。如果我们显式设置`enabled = true`，然后`enable`变量是`true`.如果我们隐式设置`enabled = undefined`，然后`enable`变量是`true`.如果我们显式设置`enabled = false`，然后`enable`变量仍然`true`!我们的目的是*违约*的值`true`，但实际上我们强制使用了值。在这种情况下，修复方法是非常明确地说明我们期望的值：

```js
function Component(props) {
  const enable = props.enabled !== false;
  // …
}
```

我们看到这种错误在每个虚假值中都会弹出。这很容易成为一个可选字符串（其中空字符串`''`被视为有效输入），或可选数字（其中`0`被视为有效输入）。这是一个非常常见的问题，我们现在引入了 nullish 合并运算符来处理这种默认值赋值：

```js
function Component(props) {
  const enable = props.enabled ?? true;
  // …
}
```

空合并运算符 （`??`） 的行为与`||`运算符，除了我们在评估运算符时不使用“truthy”。相反，我们使用“nullish”的定义，意思是“是严格等于的值`null`或`undefined`".所以想象一下表达式`lhs ?? rhs`：如果`lhs`不为空，它评估为`lhs`.否则，其计算结果为`rhs`.

明确地说，这意味着值`false`,`0`,`NaN`和空字符串`''`都是非空的虚假值。当这些虚假但非空的值是`lhs ?? rhs`，表达式的计算结果为它们，而不是右侧。虫子走了！

```js
false ?? true;   // => false
0 ?? 1;          // => 0
'' ?? 'default'; // => ''

null ?? [];      // => []
undefined ?? []; // => []
```

## 解构时的默认赋值怎么办？{ #destructuring }

您可能已经注意到，最后一个代码示例也可以通过在对象取消结构中使用默认赋值来修复：

```js
function Component(props) {
  const {
    enabled: enable = true,
  } = props;
  // …
}
```

它有点拗口，但仍然完全有效的JavaScript。但是，它使用的语义略有不同。对象取消结构内的默认赋值检查属性是否严格等于`undefined`，如果是这样，则默认分配。

但这些严格的平等测试只针对`undefined`并不总是可取的，并且要执行析构的对象并不总是可用的。例如，也许你想默认函数的返回值（没有要取消结构的对象）。或者，该函数可能返回`null`（这在 DOM API 中很常见）。这些是您希望达到空合并的时间：

```js
// Concise nullish coalescing
const link = document.querySelector('link') ?? document.createElement('link');

// Default assignment destructure with boilerplate
const {
  link = document.createElement('link'),
} = {
  link: document.querySelector('link') || undefined
};
```

此外，某些新功能，如[可选链接](/features/optional-chaining)不要完全与解构一起工作。由于析构需要一个对象，因此必须保护析构，以防可选链返回`undefined`而不是对象。对于空合并，我们没有这样的问题：

```js
// Optional chaining and nullish coalescing in tandem
const link = obj.deep?.container.link ?? document.createElement('link');

// Default assignment destructure with optional chaining
const {
  link = document.createElement('link'),
} = (obj.deep?.container || {});
```

## 混合和匹配运算符

语言设计是困难的，我们并不总是能够在开发人员的意图中没有一定程度的歧义的情况下创建新的运算符。如果您曾经混合过`&&`和`||`运算符在一起，您可能自己也遇到了这种模糊性。想象一下表达式`lhs && middle || rhs`.在JavaScript中，这实际上与表达式的解析相同`(lhs && middle) || rhs`.现在想象一下表达式`lhs || middle && rhs`.这个实际上被解析为相同的`lhs || (middle && rhs)`.

您可能会看到`&&`运算符的左侧和右侧的优先级高于`||`运算符，表示隐含的括号将`&&`而不是`||`.在设计`??`运算符，我们必须决定优先级。它可以有：

1.  优先级低于两者`&&`和`||`
2.  低于`&&`但高于`||`
3.  优先级高于两者`&&`和`||`

对于这些优先级定义中的每一个，我们都必须通过四个可能的测试用例运行它：

1.  `lhs && middle ?? rhs`
2.  `lhs ?? middle && rhs`
3.  `lhs || middle ?? rhs`
4.  `lhs ?? middle || rhs`

在每个测试表达式中，我们必须确定隐式括号所属的位置。如果他们没有完全按照开发人员的意图包装表达式，那么我们的代码就会写得很糟糕。遗憾的是，无论我们选择哪个优先级别，其中一个测试表达式都可能违反开发人员的意图。

最后，我们决定在混合时需要显式括号`??`和 （`&&`或`||`）（请注意，我的括号分组是明确的！元笑话！如果混合，则必须将其中一个运算符组括在括号中，否则会出现语法错误。

```js
// Explicit parentheses groups are required to mix
(lhs && middle) ?? rhs;
lhs && (middle ?? rhs);

(lhs ?? middle) && rhs;
lhs ?? (middle && rhs);

(lhs || middle) ?? rhs;
lhs || (middle ?? rhs);

(lhs ?? middle) || rhs;
lhs ?? (middle || rhs);
```

这样，语言解析器始终与开发人员的预期相匹配。任何后来阅读代码的人也可以立即理解它。好！

## 告诉我关于`document.all`{ #document.all }

[`document.all`](https://developer.mozilla.org/en-US/docs/Web/API/Document/all)是一个你永远不应该使用的特殊值。但是，如果您确实使用它，最好知道它如何与“真实”和“虚无”相互作用。

`document.all`是一个类似数组的对象，这意味着它具有数组和长度等索引属性。物体通常是真实的 - 但令人惊讶的是，`document.all`假装是虚假的价值！事实上，它松散地等于两者`null`和`undefined`（这通常意味着它根本不能有属性）。

使用时`document.all`与`&&`或`||`，它假装是虚假的。但是，它并不严格等于`null`也不`undefined`，所以它不是空的。因此，当使用时`document.all`跟`??`，它的行为就像任何其他对象一样。

```js
document.all || true; // => true
document.all ?? true; // => HTMLAllCollection[]
```

## 支持空合并 { #support }

<feature-support chrome="80 https://bugs.chromium.org/p/v8/issues/detail?id=9547"
              firefox="72 https://bugzilla.mozilla.org/show_bug.cgi?id=1566141"
              safari="13.1 https://webkit.org/blog/10247/new-webkit-features-in-safari-13-1/"
              nodejs="14 https://medium.com/@nodejs/node-js-version-14-available-now-8170d384567e"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-nullish-coalescing-operator"></feature-support>
