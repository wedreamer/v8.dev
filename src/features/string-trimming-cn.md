***

标题： '`String.prototype.trimStart`和`String.prototype.trimEnd`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-03-26
    标签：
*   ECMAScript
*   ES2019
    描述： 'ES2019 引入了 String.prototype.trimStart（） 和 String.prototype.trimEnd（）。'

***

ES2019介绍[`String.prototype.trimStart()`和`String.prototype.trimEnd()`](https://github.com/tc39/proposal-string-left-right-trim):

```js
const string = '  hello world  ';
string.trimStart();
// → 'hello world  '
string.trimEnd();
// → '  hello world'
string.trim(); // ES5
// → 'hello world'
```

此功能以前通过非标准提供`trimLeft()`和`trimRight()`方法，这些方法仍保留为新方法的别名，以实现向后兼容性。

```js
const string = '  hello world  ';
string.trimStart();
// → 'hello world  '
string.trimLeft();
// → 'hello world  '
string.trimEnd();
// → '  hello world'
string.trimRight();
// → '  hello world'
string.trim(); // ES5
// → 'hello world'
```

## `String.prototype.trim{Start,End}`支持 { #support }

<feature-support chrome="66 /blog/v8-release-66#string-trimming"
              firefox="61"
              safari="12"
              nodejs="8"
              babel="yes https://github.com/zloirock/core-js#ecmascript-string-and-regexp"></feature-support>
