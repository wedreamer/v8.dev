***

title： 'Subsume JSON a.k.a. JSON ⊂ ECMAScript'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-08-14
    标签：
*   ES2019
    描述： 'JSON 现在是 ECMAScript 的语法子集。
    推文：“1161649929904885762”

***

跟[这*JSON ⊂ ECMAScript*建议](https://github.com/tc39/proposal-json-superset)，JSON 成为 ECMAScript 的语法子集。如果您感到惊讶的是，情况并非如此，那么您并不孤单！

## 旧的 ES2018 行为 { #old }

在 ES2018 中，ECMAScript 字符串文本不能包含未转义的 U+2028 行分隔符和 U+2029 段落分隔符，因为它们即使在该上下文中也被视为行终止符：

```js
// A string containing a raw U+2028 character.
const LS = ' ';
// → ES2018: SyntaxError

// A string containing a raw U+2029 character, produced by `eval`:
const PS = eval('"\u2029"');
// → ES2018: SyntaxError
```

这是有问题的，因为JSON字符串*能*包含这些字符。因此，开发人员在将有效的 JSON 嵌入到 ECMAScript 程序中以处理这些字符时，必须实现专门的后处理逻辑。如果没有这样的逻辑，代码就会有微妙的错误，甚至[安全问题](#security)!

## 新行为 { #new }

在 ES2019 中，字符串文本现在可以包含原始 U+2028 和 U+2029 字符，从而消除了 ECMAScript 和 JSON 之间令人困惑的不匹配。

```js
// A string containing a raw U+2028 character.
const LS = ' ';
// → ES2018: SyntaxError
// → ES2019: no exception

// A string containing a raw U+2029 character, produced by `eval`:
const PS = eval('"\u2029"');
// → ES2018: SyntaxError
// → ES2019: no exception
```

这个小小的改进极大地简化了开发人员的心智模型（少了一个需要记住的边缘情况！），并减少了在将有效的 JSON 嵌入到 ECMAScript 程序中时对专用后处理逻辑的需求。

## 将 JSON 嵌入到 JavaScript 程序中 { #embedding-json }

由于这项建议，`JSON.stringify`现在可用于生成有效的 ECMAScript 字符串文本、对象文本和数组文本。并且由于分离[*格式良好`JSON.stringify`*建议](/features/well-formed-json-stringify)，这些文字可以安全地用 UTF-8 和其他编码表示（如果您尝试将它们写入磁盘上的文件，这将非常有用）。这对于元编程用例非常有用，例如动态创建JavaScript源代码并将其写入磁盘。

下面是一个示例，它利用 JSON 语法现在是 ECMAScript 的子集，创建嵌入给定数据对象的有效 JavaScript 程序：

```js
// A JavaScript object (or array, or string) representing some data.
const data = {
  LineTerminators: '\n\r  ',
  // Note: the string contains 4 characters: '\n\r\u2028\u2029'.
};

// Turn the data into its JSON-stringified form. Thanks to JSON ⊂
// ECMAScript, the output of `JSON.stringify` is guaranteed to be
// a syntactically valid ECMAScript literal:
const jsObjectLiteral = JSON.stringify(data);

// Create a valid ECMAScript program that embeds the data as an object
// literal.
const program = `const data = ${ jsObjectLiteral };`;
// → 'const data = {"LineTerminators":"…"};'
// (Additional escaping is needed if the target is an inline <script>.)

// Write a file containing the ECMAScript program to disk.
saveToDisk(filePath, program);
```

上面的脚本生成以下代码，该代码的计算结果为等效对象：

```js
const data = {"LineTerminators":"\n\r  "};
```

## 将 JSON 嵌入到 JavaScript 程序中`JSON.parse`{ #embedding-json-parse }

如 中所述[*JSON 的成本*](/blog/cost-of-javascript-2019#json)，而不是将数据内联为 JavaScript 对象文本，如下所示：

```js
const data = { foo: 42, bar: 1337 }; // 🐌
```

...数据可以以 JSON 字符串化形式表示，然后在运行时进行 JSON 解析，以提高大型对象 （10 kB+） 的性能：

```js
const data = JSON.parse('{"foo":42,"bar":1337}'); // 🚀
```

下面是一个示例实现：

```js
// A JavaScript object (or array, or string) representing some data.
const data = {
  LineTerminators: '\n\r  ',
  // Note: the string contains 4 characters: '\n\r\u2028\u2029'.
};

// Turn the data into its JSON-stringified form.
const json = JSON.stringify(data);

// Now, we want to insert the JSON into a script body as a JavaScript
// string literal per https://v8.dev/blog/cost-of-javascript-2019#json,
// escaping special characters like `"` in the data.
// Thanks to JSON ⊂ ECMAScript, the output of `JSON.stringify` is
// guaranteed to be a syntactically valid ECMAScript literal:
const jsStringLiteral = JSON.stringify(json);
// Create a valid ECMAScript program that embeds the JavaScript string
// literal representing the JSON data within a `JSON.parse` call.
const program = `const data = JSON.parse(${ jsStringLiteral });`;
// → 'const data = JSON.parse("…");'
// (Additional escaping is needed if the target is an inline <script>.)

// Write a file containing the ECMAScript program to disk.
saveToDisk(filePath, program);
```

上面的脚本生成以下代码，该代码的计算结果为等效对象：

```js
const data = JSON.parse("{\"LineTerminators\":\"\\n\\r  \"}");
```

[谷歌的基准比较`JSON.parse`使用 JavaScript 对象文本](https://github.com/GoogleChromeLabs/json-parse-benchmark)在其生成步骤中利用此技术。Chrome DevTools的“copy as JS”功能已经[显著简化](https://chromium-review.googlesource.com/c/chromium/src/+/1464719/9/third_party/blink/renderer/devtools/front_end/elements/DOMPath.js)通过采用类似的技术。

## 关于安全性的说明 { #security }

JSON ⊂ ECMAScript 减少了 JSON 和 ECMAScript 在字符串文本的情况下的不匹配。由于字符串文本可以出现在其他 JSON 支持的数据结构（如对象和数组）中，因此它也解决了这些情况，如上面的代码示例所示。

但是，U+2028 和 U+2029 在 ECMAScript 语法的其他部分中仍被视为行终止符字符。这意味着在某些情况下，将JSON注入JavaScript程序是不安全的。考虑此示例，其中服务器在运行 HTML 响应后将一些用户提供的内容注入到 HTML 响应中。`JSON.stringify()`:

```ejs
<script>
  // Debug info:
  // User-Agent: <%= JSON.stringify(ua) %>
</script>
```

请注意，结果`JSON.stringify`注入到脚本中的单行注释中。

当像上面的例子中使用时，`JSON.stringify()`保证返回一行。问题在于，什么构成了“单线”。[JSON 和 ECMAScript 之间的差异](https://speakerdeck.com/mathiasbynens/hacking-with-unicode?slide=136).如果`ua`包含一个未转义的 U+2028 或 U+2029 字符，我们脱离单行注释并执行其余的`ua`作为 JavaScript 源代码：

```html
<script>
  // Debug info:
  // User-Agent: "User-supplied string<U+2028>  alert('XSS');//"
</script>
<!-- …is equivalent to: -->
<script>
  // Debug info:
  // User-Agent: "User-supplied string
  alert('XSS');//"
</script>
```

：：：备注
**注意：**在上面的示例中，原始的未转义 U+2028 字符表示为`<U+2028>`使其更易于遵循。
:::

JSON ⊂ ECMAScript 在这里没有帮助，因为它只影响字符串文本 — 在这种情况下，`JSON.stringify`的输出被注入到一个位置，它不会直接生成 JavaScript 字符串文本。

除非对这两个字符引入特殊的后处理，否则上述代码片段会出现一个跨站点脚本漏洞（XSS）！

：：：备注
**注意：**根据上下文，对用户控制的输入进行后处理以转义任何特殊字符序列至关重要。在这种特殊情况下，我们将注入`<script>`标记之间，因此我们必须（也）[逃`</script`,`<script`和`<!-​-`](https://mathiasbynens.be/notes/etago#recommendations).
:::

## JSON ⊂ ECMAScript support { #support }

<feature-support chrome="66 /blog/v8-release-66#json-ecmascript"
              firefox="yes"
              safari="yes"
              nodejs="10"
              babel="yes https://github.com/babel/babel/tree/master/packages/babel-plugin-proposal-json-strings"></feature-support>
