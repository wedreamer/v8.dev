***

标题： '`String.prototype.matchAll`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-02-02
    标签：
*   ECMAScript
*   ES2020
*   io19
    描述： 'String.prototype.matchAll 可以很容易地循环访问给定正则表达式生成的所有匹配对象。

***

通常对字符串重复应用相同的正则表达式以获取所有匹配项。在某种程度上，今天通过使用`String#match`方法。

在此示例中，我们查找仅由十六进制数字组成的所有单词，然后记录每个匹配项：

```js
const string = 'Magic hex numbers: DEADBEEF CAFE';
const regex = /\b\p{ASCII_Hex_Digit}+\b/gu;
for (const match of string.match(regex)) {
  console.log(match);
}

// Output:
//
// 'DEADBEEF'
// 'CAFE'
```

但是，这只会为您提供*子字符串*那场比赛。通常，您不仅需要子字符串，还需要其他信息，例如每个子字符串的索引或每个匹配项中的捕获组。

通过编写自己的循环并自己跟踪匹配对象已经可以实现这一点，但这有点烦人，而且不是很方便：

```js
const string = 'Magic hex numbers: DEADBEEF CAFE';
const regex = /\b\p{ASCII_Hex_Digit}+\b/gu;
let match;
while (match = regex.exec(string)) {
  console.log(match);
}

// Output:
//
// [ 'DEADBEEF', index: 19, input: 'Magic hex numbers: DEADBEEF CAFE' ]
// [ 'CAFE',     index: 28, input: 'Magic hex numbers: DEADBEEF CAFE' ]
```

新`String#matchAll`API使这比以往任何时候都更容易：您现在可以编写一个简单的`for`-`of`循环以获取所有匹配对象。

```js
const string = 'Magic hex numbers: DEADBEEF CAFE';
const regex = /\b\p{ASCII_Hex_Digit}+\b/gu;
for (const match of string.matchAll(regex)) {
  console.log(match);
}

// Output:
//
// [ 'DEADBEEF', index: 19, input: 'Magic hex numbers: DEADBEEF CAFE' ]
// [ 'CAFE',     index: 28, input: 'Magic hex numbers: DEADBEEF CAFE' ]
```

`String#matchAll`对于具有捕获组的正则表达式特别有用。它为您提供了每个匹配项的完整信息，包括捕获组。

```js
const string = 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev';
const regex = /\b(?<owner>[a-z0-9]+)\/(?<repo>[a-z0-9\.]+)\b/g;
for (const match of string.matchAll(regex)) {
  console.log(`${match[0]} at ${match.index} with '${match.input}'`);
  console.log(`→ owner: ${match.groups.owner}`);
  console.log(`→ repo: ${match.groups.repo}`);
}

// Output:
//
// tc39/ecma262 at 23 with 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev'
// → owner: tc39
// → repo: ecma262
// v8/v8.dev at 36 with 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev'
// → owner: v8
// → repo: v8.dev
```

一般的想法是，你只是写一个简单的`for`-`of`循环，以及`String#matchAll`为您照顾其余的。

：：：备注
**注意：**顾名思义，`String#matchAll`旨在循环访问*都*匹配对象。因此，它应该与全局正则表达式一起使用，即那些具有`g`标志集，因为任何非全局正则表达式都只会生成一个匹配项（最多）。叫`matchAll`使用非全局正则表达式的结果为`TypeError`例外。
:::

## `String.prototype.matchAll`支持 { #support }

<feature-support chrome="73 /blog/v8-release-73#string.prototype.matchall"
              firefox="67"
              safari="13"
              nodejs="12"
              babel="yes https://github.com/zloirock/core-js#ecmascript-string-and-regexp"></feature-support>
