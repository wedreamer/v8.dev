***

标题： “数字分隔符”
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-05-28
    标签：
*   ECMAScript
*   ES2021
*   io19
    描述：“JavaScript现在支持下划线作为数字文本中的分隔符，从而提高了源代码的可读性和可维护性。
    推文：“1129073383931559936”

***

较大的数字文字很难使人眼快速解析，特别是当有很多重复的数字时：

```js
1000000000000
   1019436871.42
```

为了提高可读性，[一个新的 JavaScript 语言特性](https://github.com/tc39/proposal-numeric-separator)启用下划线作为数字文本中的分隔符。因此，现在可以重写上述内容以对每千个数字进行分组，例如：

```js
1_000_000_000_000
    1_019_436_871.42
```

现在更容易分辨出第一个数字是一万亿，第二个数字是10亿。

数字分隔符有助于提高各种数字文本的可读性：

```js
// A decimal integer literal with its digits grouped per thousand:
1_000_000_000_000
// A decimal literal with its digits grouped per thousand:
1_000_000.220_720
// A binary integer literal with its bits grouped per octet:
0b01010110_00111000
// A binary integer literal with its bits grouped per nibble:
0b0101_0110_0011_1000
// A hexadecimal integer literal with its digits grouped by byte:
0x40_76_38_6A_73
// A BigInt literal with its digits grouped per thousand:
4_642_473_943_484_686_707n
```

它们甚至适用于八进制整数文本（尽管[我想不出一个例子](https://github.com/tc39/proposal-numeric-separator/issues/44)其中分隔符为此类文本提供值）：

```js
// A numeric separator in an octal integer literal: 🤷‍♀️
0o123_456
```

请注意，JavaScript 还为八进制文本保留了一个语法，但没有显式`0o`前缀。例如`017 === 0o17`.此语法在严格模式或模块中不受支持，不应在新式代码中使用。因此，这些文本不支持数字分隔符。用`0o17`-样式文字代替。

## 支持数字分隔符 { #support }

<feature-support chrome="75 /blog/v8-release-75#numeric-separators"
              firefox="70 https://hacks.mozilla.org/2019/10/firefox-70-a-bountiful-release-for-all/"
              safari="13"
              nodejs="12.5.0 https://nodejs.org/en/blog/release/v12.5.0/"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-numeric-separator"></feature-support>
