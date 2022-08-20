***

标题： '格式良好`JSON.stringify`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-09-11
    标签：
*   ECMAScript
*   ES2019
    描述： 'JSON.stringify现在为孤独的代理输出转义序列，使其输出有效的Unicode（并且可以用UTF-8表示）。

***

`JSON.stringify`之前指定为在输入包含任何单独的代理项时返回格式错误的 Unicode 字符串：

```js
JSON.stringify('\uD800');
// → '"�"'
```

[“形式良好”`JSON.stringify`“提案](https://github.com/tc39/proposal-well-formed-stringify)变化`JSON.stringify`因此，它为单独的代理项输出转义序列，使其输出有效的Unicode（并且可以用UTF-8表示）：

```js
JSON.stringify('\uD800');
// → '"\\ud800"'
```

请注意，`JSON.parse(stringified)`仍然产生与以前相同的结果。

这个功能是一个小的修复，在JavaScript中早就应该了。作为JavaScript开发人员，这少了一件需要担心的事情。结合[*JSON ⊂ ECMAScript*](/features/subsume-json)，它允许将 JSON 字符串化数据作为文本安全地嵌入到 JavaScript 程序中，并以任何与 Unicode 兼容的编码（例如 UTF-8）将生成的代码写入磁盘。这对于[元编程用例](/features/subsume-json#embedding-json).

## 功能支持 { #support }

<feature-support chrome="72 /blog/v8-release-72#well-formed-json.stringify"
              firefox="64"
              safari="12.1"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-json"></feature-support>
