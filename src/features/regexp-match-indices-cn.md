***

标题： “正则表达式匹配索引”
作者： '玛雅·阿米拉诺娃 （[@Zmayski](https://twitter.com/Zmayski)），定期表达新功能'
化身：

*   '玛雅-军队'
    日期： 2019-12-17
    标签：
*   ECMAScript
*   节点.js 16
    描述： '正则表达式匹配索引提供`start`和`end`每个匹配捕获组的索引。
    推文：“1206970814400270338”

***

JavaScript现在配备了一个新的正则表达式增强功能，称为“匹配索引”。想象一下，您希望在 JavaScript 代码中找到与保留字一致的无效变量名称，并在变量名称下输出插入符号和“下划线”，例如：

```js
const function = foo;
      ^------- Invalid variable name
```

在上面的例子中，`function`是保留字，不能用作变量名称。为此，我们可以编写以下函数：

```js
function displayError(text, message) {
  const re = /\b(continue|function|break|for|if)\b/d;
  const match = text.match(re);
  // Index `1` corresponds to the first capture group.
  const [start, end] = match.indices[1];
  const error = ' '.repeat(start) + // Adjust the caret position.
    '^' +
    '-'.repeat(end - start - 1) +   // Append the underline.
    ' ' + message;                  // Append the message.
  console.log(text);
  console.log(error);
}

const code = 'const function = foo;'; // faulty code
displayError(code, 'Invalid variable name');
```

：：：备注
**注意：**为简单起见，上面的示例仅包含一些 JavaScript[保留字](https://mathiasbynens.be/notes/reserved-keywords).
:::

简而言之，新的`indices`数组存储每个匹配的捕获组的开始和结束位置。当源正则表达式使用`/d`生成正则表达式匹配对象的所有内置的标志，包括`RegExp#exec`,`String#match`和[`String#matchAll`](https://v8.dev/features/string-matchall).

如果您对它的工作原理感兴趣，请继续阅读。

## 动机

让我们转到一个更复杂的示例，并考虑如何解决解析编程语言的任务（例如[TypeScript compiler](https://github.com/microsoft/TypeScript/tree/master/src/compiler)does） — 首先将输入源代码拆分为标记，然后为这些标记提供语法结构。如果用户编写了一些语法不正确的代码，您可能希望向他们显示有意义的错误，最好指向首次遇到问题代码的位置。例如，给定以下代码片段：

```js
let foo = 42;
// some other code
let foo = 1337;
```

我们希望向程序员显示一个错误，如下所示：

```js
let foo = 1337;
    ^
SyntaxError: Identifier 'foo' has already been declared
```

为了实现这一点，我们需要一些构建块，其中第一个是识别TypeScript标识符。然后，我们将专注于查明错误发生的确切位置。让我们考虑以下示例，使用正则表达式来判断字符串是否为有效标识符：

```js
function isIdentifier(name) {
  const re = /^[a-zA-Z_$][0-9a-zA-Z_$]*$/;
  return re.exec(name) !== null;
}
```

：：：备注
**注意：**现实世界的解析器可以利用新引入的[属性在正则表达式中转义](https://github.com/tc39/proposal-regexp-unicode-property-escapes#other-examples)并使用以下正则表达式来匹配所有有效的 ECMAScript 标识符名称：

```js
const re = /^[$_\p{ID_Start}][$_\u200C\u200D\p{ID_Continue}]*$/u;
```

为简单起见，让我们坚持使用前面的正则表达式，它仅匹配拉丁字符、数字和下划线。
:::

如果我们遇到类似上面的变量声明的错误，并希望将确切的位置打印给用户，我们可能希望从上面扩展正则表达式并使用类似的函数：

```js
function getDeclarationPosition(source) {
  const re = /(let|const|var)\s+([a-zA-Z_$][0-9a-zA-Z_$]*)/;
  const match = re.exec(source);
  if (!match) return -1;
  return match.index;
}
```

可以使用`index`返回的匹配对象上的属性`RegExp.prototype.exec`，返回整个匹配项的起始位置。但是，对于上述用例，您通常希望使用（可能多个）捕获组。直到最近，JavaScript 还没有公开由捕获组匹配的子字符串开始和结束的索引。

## 正则表达式匹配指数说明

理想情况下，我们希望在变量名称的位置打印错误，而不是在`let`/`const`关键字（如上面的示例所示）。但为此，我们需要找到具有索引的捕获组的位置`2`.（索引`1`指`(let|const|var)`捕获组和`0`指整个匹配项。

如上所述，[新的 JavaScript 功能](https://github.com/tc39/proposal-regexp-match-indices)添加一个`indices`结果（子字符串数组）上的属性`RegExp.prototype.exec()`.让我们从上面增强我们的示例以利用这个新属性：

```js
function getVariablePosition(source) {
  // Notice the `d` flag, which enables `match.indices`
  const re = /(let|const|var)\s+([a-zA-Z_$][0-9a-zA-Z_$]*)/d;
  const match = re.exec(source);
  if (!match) return undefined;
  return match.indices[2];
}
getVariablePosition('let foo');
// → [4, 7]
```

此示例返回数组`[4, 7]`，即`[start, end)`具有索引的组中匹配的子字符串的位置`2`.根据这些信息，我们的编译器现在可以打印所需的错误。

## 附加功能

这`indices`对象还包含一个`groups`属性，可按[命名捕获组](https://mathiasbynens.be/notes/es-regexp-proposals#named-capture-groups).使用它，上面的函数可以重写为：

```js
function getVariablePosition(source) {
  const re = /(?<keyword>let|const|var)\s+(?<id>[a-zA-Z_$][0-9a-zA-Z_$]*)/d;
  const match = re.exec(source);
  if (!match) return -1;
  return match.indices.groups.id;
}
getVariablePosition('let foo');
```

## 支持正则表达式匹配索引

<feature-support chrome="90 https://bugs.chromium.org/p/v8/issues/detail?id=9548"
              firefox="no https://bugzilla.mozilla.org/show_bug.cgi?id=1519483"
              safari="no https://bugs.webkit.org/show_bug.cgi?id=202475"
              nodejs="16"
              babel="no"></feature-support>
