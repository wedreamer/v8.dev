***

标题： '`String.prototype.replaceAll`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-11-11
    标签：
*   ECMAScript
*   ES2021
*   节点.js 16
    描述： 'JavaScript 现在具有一流的支持，可通过新的全局子字符串替换`String.prototype.replaceAll`API.'
    推文：“1193917549060280320”

***

如果您曾经在JavaScript中处理过字符串，那么您很可能会遇到`String#replace`方法。`String.prototype.replace(searchValue, replacement)`根据您指定的参数返回一个字符串，其中替换了一些匹配项：

```js
'abc'.replace('b', '_');
// → 'a_c'

'🍏🍋🍊🍓'.replace('🍏', '🥭');
// → '🥭🍋🍊🍓'
```

一个常见的用例是替换*都*给定子字符串的实例。然而`String#replace`不直接解决此用例。什么时候`searchValue`是一个字符串，只有第一次出现的子字符串才会被替换：

```js
'aabbcc'.replace('b', '_');
// → 'aa_bcc'

'🍏🍏🍋🍋🍊🍊🍓🍓'.replace('🍏', '🥭');
// → '🥭🍏🍋🍋🍊🍊🍓🍓'
```

为了解决这个问题，开发人员通常将搜索字符串转换为具有全局（`g`） 标志。这边`String#replace`确实替换*都*比赛：

```js
'aabbcc'.replace(/b/g, '_');
// → 'aa__cc'

'🍏🍏🍋🍋🍊🍊🍓🍓'.replace(/🍏/g, '🥭');
// → '🥭🥭🍋🍋🍊🍊🍓🍓'
```

作为开发人员，如果您真正想要的只是全局子字符串替换，那么必须进行这种字符串到正则表达式的转换是很烦人的。更重要的是，这种转换容易出错，并且是错误的常见来源！请考虑以下示例：

```js
const queryString = 'q=query+string+parameters';

queryString.replace('+', ' ');
// → 'q=query string+parameters' ❌
// Only the first occurrence gets replaced.

queryString.replace(/+/, ' ');
// → SyntaxError: invalid regular expression ❌
// As it turns out, `+` is a special character within regexp patterns.

queryString.replace(/\+/, ' ');
// → 'q=query string+parameters' ❌
// Escaping special regexp characters makes the regexp valid, but
// this still only replaces the first occurrence of `+` in the string.

queryString.replace(/\+/g, ' ');
// → 'q=query string parameters' ✅
// Escaping special regexp characters AND using the `g` flag makes it work.
```

将字符串文本转换为`'+'`进入全局正则表达式不仅仅是删除`'`引号，将其包装成`/`斜杠，并附加`g`flag — 我们必须转义任何在正则表达式中具有特殊含义的字符。这很容易忘记，也很难正确，因为JavaScript没有提供内置的机制来逃避正则表达式模式。

另一种解决方法是将`String#split`跟`Array#join`:

```js
const queryString = 'q=query+string+parameters';
queryString.split('+').join(' ');
// → 'q=query string parameters'
```

这种方法避免了任何转义，但附带了将字符串拆分为一组部分的开销，只是为了将其粘合在一起。

显然，这些解决方法都不是理想的。如果像全局子字符串替换这样的基本操作在JavaScript中很简单，那不是很好吗？

## `String.prototype.replaceAll`

新`String#replaceAll`方法解决了这些问题，并提供了一种简单的机制来执行全局子字符串替换：

```js
'aabbcc'.replaceAll('b', '_');
// → 'aa__cc'

'🍏🍏🍋🍋🍊🍊🍓🍓'.replaceAll('🍏', '🥭');
// → '🥭🥭🍋🍋🍊🍊🍓🍓'

const queryString = 'q=query+string+parameters';
queryString.replaceAll('+', ' ');
// → 'q=query string parameters'
```

为了与语言中预先存在的 API 保持一致，`String.prototype.replaceAll(searchValue, replacement)`行为完全符合`String.prototype.replace(searchValue, replacement)`，但以下两个例外：

1.  如果`searchValue`是一个字符串，则`String#replace`仅替换子字符串的第一个匹配项，而`String#replaceAll`取代*都*事件。
2.  如果`searchValue`是非全局正则表达式，则`String#replace`仅替换单个匹配项，类似于它对字符串的行为方式。`String#replaceAll`另一方面，在这种情况下会抛出一个异常，因为这可能是一个错误：如果你真的想“替换所有”匹配项，你会使用全局正则表达式;如果您只想替换单个匹配项，则可以使用`String#replace`.

新功能的重要部分在于第一项。`String.prototype.replaceAll`通过对全局子字符串替换的一流支持来丰富 JavaScript，而无需正则表达式或其他解决方法。

## 关于特殊替换模式的说明 { #special模式 }

值得一提的是：两者兼而有之`replace`和`replaceAll`支持[特殊替换模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#Specifying_a_string_as_a_parameter).虽然这些与正则表达式结合使用最有用，但其中一些（`$$`,`$&`,`` $`  ``和`$'`） 在执行简单的字符串替换时也会生效，这可能会令人惊讶：

```js
'xyz'.replaceAll('y', '$$');
// → 'x$z' (not 'x$$z')
```

如果您的替换字符串包含这些模式之一，并且您希望按原样使用它们，则可以通过使用返回字符串的替换器函数来选择退出神奇的替换行为：

```js
'xyz'.replaceAll('y', () => '$$');
// → 'x$$z'
```

## `String.prototype.replaceAll`支持 { #support }

<feature-support chrome="85 https://bugs.chromium.org/p/v8/issues/detail?id=9801"
              firefox="77 https://bugzilla.mozilla.org/show_bug.cgi?id=1608168#c8"
              safari="13.1 https://webkit.org/blog/10247/new-webkit-features-in-safari-13-1/"
              nodejs="16"
              babel="yes https://github.com/zloirock/core-js#ecmascript-string-and-regexp"></feature-support>
