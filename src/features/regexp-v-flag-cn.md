***

标题： '正则表达式`v`具有字符串的设置表示法和属性的标志
作者： '马克·戴维斯 （[@mark_e_davis](https://twitter.com/mark_e_davis)），Markus Scherer和Mathias Bynens（[@mathias](https://twitter.com/mathias))'
化身：

*   “马克-戴维斯”
*   'markus-scherer'
*   'mathias-bynens'
    日期： 2022-06-27
    标签：
*   ECMAScript
    描述： '新的正则表达式`v`标志启用`unicodeSets`模式，解锁对扩展字符类的支持，包括字符串的 Unicode 属性、集合表示法和改进的不区分大小写匹配。
    推文：“1541419838513594368”

***

JavaScript 自 ECMAScript 3 （1999） 以来一直支持正则表达式。16年后，ES2015问世[统一码模式（`u`标志）](https://mathiasbynens.be/notes/es6-unicode-regex),[粘滞模式（`y`标志）](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky#description)和[这`RegExp.prototype.flags`吸气剂](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/flags).又过了三年，ES2018问世[`dotAll`模式（`s`标志）](https://mathiasbynens.be/notes/es-regexp-proposals#dotAll),[查看断言](https://mathiasbynens.be/notes/es-regexp-proposals#lookbehinds),[命名捕获组](https://mathiasbynens.be/notes/es-regexp-proposals#named-capture-groups)和[Unicode 字符属性转义](https://mathiasbynens.be/notes/es-unicode-property-escapes).在ES2020中，[`String.prototype.matchAll`](https://v8.dev/features/string-matchall)使使用正则表达式变得更加容易。JavaScript正则表达式已经走过了漫长的道路，并且仍在改进。

这方面的最新例子是[新的`unicodeSets`模式，使用`v`旗](https://github.com/tc39/proposal-regexp-v-flag).此新模式可解锁对*扩展字符类*，包括以下功能：

*   [字符串的 Unicode 属性](/features/regexp-v-flag#unicode-properties-of-strings)
*   [设置表示法 + 字符串文本语法](/features/regexp-v-flag#set-notation)
*   [改进了不区分大小写的匹配](/features/regexp-v-flag#ignoreCase)

本文将深入探讨其中的每一项。但首先要做的是 — 以下是使用新标志的方法：

```js
const re = /…/v;
```

这`v`标志可以与现有的正则表达式标志结合使用，但有一个值得注意的例外。这`v`标志启用所有好的部分`u`标记，但具有其他功能和改进 — 其中一些与`u`旗。关键`v`是一个完全独立的模式`u`而不是互补的。因此，`v`和`u`标志不能组合 — 尝试在同一正则表达式上使用这两个标志会导致错误。唯一有效的选项是：使用`u`，或使用`v`，或两者都不使用`u`也不`v`.但自从`v`是功能最齐全的选项，选择很容易做出...

让我们深入了解新功能！

## 字符串的 Unicode 属性

Unicode 标准为每个符号分配各种属性和属性值。例如，若要获取希腊语脚本中使用的符号集，请在 Unicode 数据库中搜索其`Script_Extensions`属性值包括`Greek`.

ES2018 Unicode 字符属性转义使得在 ECMAScript 正则表达式中以本机方式访问这些 Unicode 字符属性成为可能。例如，模式`\p{Script_Extensions=Greek}`匹配希腊语脚本中使用的每个符号：

```js
const regexGreekSymbol = /\p{Script_Extensions=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

根据定义，Unicode 字符属性扩展到一组代码点，因此可以转译为包含它们单独匹配的代码点的字符类。例如`\p{ASCII_Hex_Digit}`等效于`[0-9A-Fa-f]`：它一次只匹配一个 Unicode 字符/码位。在某些情况下，这是不够的：

```js
// Unicode defines a character property named “Emoji”.
const re = /^\p{Emoji}$/u;

// Match an emoji that consists of just 1 code point:
re.test('⚽'); // '\u26BD'
// → true ✅

// Match an emoji that consists of multiple code points:
re.test('👨🏾‍⚕️'); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → false ❌
```

在上面的示例中，正则表达式与表情符号不匹配，👨🏾 ⚕️因为它恰好由多个码位组成，并且`Emoji`是一个统一码*字符*财产。

幸运的是，Unicode标准还定义了几个[字符串的属性](https://www.unicode.org/reports/tr18/#domain_of_properties).此类属性扩展为一组字符串，每个字符串包含一个或多个代码点。在正则表达式中，字符串的属性转换为一组替代项。为了说明这一点，想象一个适用于字符串的 Unicode 属性`'a'`,`'b'`,`'c'`,`'W'`,`'xy'`和`'xyz'`.此属性转换为以下任一正则表达式模式（使用交替）：`xyz|xy|a|b|c|W`或`xyz|xy|[a-cW]`.（最长的字符串优先，以便前缀像`'xy'`不隐藏较长的字符串，如`'xyz'`.)与现有的 Unicode 属性转义不同，此模式可以匹配多字符字符串。下面是正在使用的字符串的属性示例：

```js
const re = /^\p{RGI_Emoji}$/v;

// Match an emoji that consists of just 1 code point:
re.test('⚽'); // '\u26BD'
// → true ✅

// Match an emoji that consists of multiple code points:
re.test('👨🏾‍⚕️'); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → true ✅
```

此代码片段引用字符串的属性`RGI_Emoji`，Unicode 将其定义为“建议用于常规交换的所有有效表情符号（字符和序列）的子集”。有了这个，我们现在可以匹配表情符号，无论它们在引擎盖下包含多少个代码点！

这`v`标记允许从一开始就支持字符串的以下 Unicode 属性：

*   `Basic_Emoji`
*   `Emoji_Keycap_Sequence`
*   `RGI_Emoji_Modifier_Sequence`
*   `RGI_Emoji_Flag_Sequence`
*   `RGI_Emoji_Tag_Sequence`
*   `RGI_Emoji_ZWJ_Sequence`
*   `RGI_Emoji`

此受支持属性列表将来可能会随着 Unicode 标准定义字符串的其他属性而增长。尽管字符串的所有当前属性恰好都与表情符号相关，但字符串的未来属性可能服务于完全不同的用例。

：：：备注
**注意：**尽管字符串的属性当前已在新的`v`旗[我们计划最终使它们在`u`模式以及](https://github.com/tc39/proposal-regexp-v-flag/issues/49).
:::

## 设置表示法 + 字符串文本语法 { #set表示法 }

使用时`\p{…}`转义（无论是字符属性还是字符串的新属性），执行差分/减法或交集都很有用。随着`v`标志，字符类现在可以嵌套，并且这些集合操作现在可以在其中执行，而不是使用相邻的查找或查找后置断言或表示计算范围的冗长字符类。

### 差/减法`--`{ #difference }

语法`A--B`可用于匹配字符串*在`A`但不在`B`*，又名差/减法。

例如，如果要匹配除字母之外的所有希腊语符号，该怎么办`π`?使用集合表示法，解决这个问题是微不足道的：

```js
/[\p{Script_Extensions=Greek}--π]/v.test('π'); // → false
```

通过使用`--`对于差分/减法，正则表达式引擎为您完成艰苦的工作，同时保持代码的可读性和可维护性。

如果我们想要减去字符集，而不是单个字符，该怎么办`α`,`β`和`γ`?没问题 — 我们可以使用嵌套字符类并减去其内容：

```js
/[\p{Script_Extensions=Greek}--[αβγ]]/v.test('α'); // → false
/[\p{Script_Extensions=Greek}--[α-γ]]/v.test('β'); // → false
```

另一个示例是匹配非 ASCII 数字，例如稍后将它们转换为 ASCII 数字：

```js
/[\p{Decimal_Number}--[0-9]]/v.test('𑜹'); // → true
/[\p{Decimal_Number}--[0-9]]/v.test('4'); // → false
```

集合表示法也可以与字符串的新属性一起使用：

```js
// Note: 🏴󠁧󠁢󠁳󠁣󠁴󠁿 consists of 7 code points.

/^\p{RGI_Emoji_Tag_Sequence}$/v.test('🏴󠁧󠁢󠁳󠁣󠁴󠁿'); // → true
/^[\p{RGI_Emoji_Tag_Sequence}--\q{🏴󠁧󠁢󠁳󠁣󠁴󠁿}]$/v.test('🏴󠁧󠁢󠁳󠁣󠁴󠁿'); // → false
```

本示例匹配任何 RGI 表情符号标记序列*除了*苏格兰国旗。注意使用`\q{…}`，这是字符类中字符串文本的另一个新语法。例如`\q{a|bc|def}`匹配字符串`a`,`bc`和`def`.没有`\q{…}`不可能减去硬编码的多字符字符串。

### 与`&&`{ #intersection }

这`A&&B`语法匹配以下字符串：*在两者中`A`和`B`*，又名交叉点。这使您可以执行匹配希腊字母之类的操作：

```js
const re = /[\p{Script_Extensions=Greek}&&\p{Letter}]/v;
// U+03C0 GREEK SMALL LETTER PI
re.test('π'); // → true
// U+1018A GREEK ZERO SIGN
re.test('𐆊'); // → false
```

匹配所有 ASCII 空格：

```js
const re = [\p{White_Space}&&\p{ASCII}];
re.test('\n'); // → true
re.test('\u2028'); // → false
```

或匹配所有蒙古语号码：

```js
const re = [\p{Script_Extensions=Mongolian}&&\p{Number}];
// U+1817 MONGOLIAN DIGIT SEVEN
re.test('᠗'); // → true
// U+1834 MONGOLIAN LETTER CHA
re.test('ᠴ'); // → false
```

### 联盟

匹配以下字符串：*在 A 或 B 中*以前已经可以通过使用字符类（如`[\p{Letter}\p{Number}]`.随着`v`标记，此功能变得更加强大，因为它现在也可以与字符串或字符串文本的属性组合：

```js
const re = /^[\p{Emoji_Keycap_Sequence}\p{ASCII}\q{🇧🇪|abc}xyz0-9]$/v;

re.test('4️⃣'); // → true
re.test('_'); // → true
re.test('🇧🇪'); // → true
re.test('abc'); // → true
re.test('x'); // → true
re.test('4'); // → true
```

此模式中的字符类组合了：

*   字符串 （`\p{Emoji_Keycap_Sequence}`)
*   字符属性 （`\p{ASCII}`)
*   多代码点字符串的字符串文本语法`🇧🇪`和`abc`
*   孤独字符的经典字符类语法`x`,`y`和`z`
*   经典字符类语法，字符范围从`0`自`9`

另一个示例是匹配所有常用的标志表情符号，无论它们是否编码为两个字母的 ISO 代码 （`RGI_Emoji_Flag_Sequence`） 或作为特殊大小写的标记序列 （`RGI_Emoji_Tag_Sequence`):

```js
const reFlag = /[\p{RGI_Emoji_Flag_Sequence}\p{RGI_Emoji_Tag_Sequence}]/v;
// A flag sequence, consisting of 2 code points (flag of Belgium):
re.test('🇧🇪'); // → true
// A tag sequence, consisting of 7 code points (flag of England):
re.test('🏴󠁧󠁢󠁥󠁮󠁧󠁿'); // → true
// A flag sequence, consisting of 2 code points (flag of Switzerland):
re.test('🇨🇭'); // → true
// A tag sequence, consisting of 7 code points (flag of Wales):
re.test('🏴󠁧󠁢󠁷󠁬󠁳󠁿'); // → true
```

## 改进了不区分大小写的匹配 { #ignoreCase }

The ES2015`u`旗帜遭受[令人困惑的不区分大小写的匹配行为](https://github.com/tc39/proposal-regexp-v-flag/issues/30).请考虑以下两个正则表达式：

```js
const re1 = /\p{Lowercase_Letter}/giu;
const re2 = /[^\P{Lowercase_Letter}]/giu;
```

第一个模式匹配所有小写字母。第二种模式使用`\P`而不是`\p`以匹配除小写字母之外的所有字符，但随后被包装在否定字符类 （`[^…]`).这两个正则表达式都不区分大小写，方法是将`i`标志 （`ignoreCase`).

直观地说，您可能希望两个正则表达式的行为相同。在实践中，它们的行为非常不同：

```js
const string = 'aAbBcC4#';

string.replaceAll(re1, 'X');
// → 'XXXXXX4#'

string.replaceAll(re2, 'X');
// → 'aAbBcC4#''
```

新`v`标志具有较少的意外行为。随着`v`标志而不是`u`标记，两种模式的行为相同：

```js
const re1 = /\p{Lowercase_Letter}/giv;
const re2 = /[^\P{Lowercase_Letter}]/giv;

const string = 'aAbBcC4#';

string.replaceAll(re1, 'X');
// → 'XXXXXX4#'

string.replaceAll(re2, 'X');
// → 'XXXXXX4#'
```

更一般地说，`v`标志使`[^\p{X}]`≍`[\P{X}]`≍`\P{X}`和`[^\P{X}]`≍`[\p{X}]`≍`\p{X}`，是否`i`标志是否已设置。

## 进一步阅读

[提案存储库](https://github.com/tc39/proposal-regexp-v-flag)包含有关这些功能及其设计决策的更多详细信息和背景。

作为我们这些 JavaScript 功能工作的一部分，我们超越了“仅仅”提出对 ECMAScript 的规范更改。我们将“字符串属性”的定义上游到[统一码 UTS#18](https://unicode.org/reports/tr18/#Notation_for_Properties_of_Strings)以便其他编程语言可以统一实现类似的功能。我们也是[提议对 HTML 标准进行更改](https://github.com/whatwg/html/pull/7908)目标是在`pattern`属性也是如此。

## 正则表达式`v`标志支持 { #support }

这`v`标志在任何 JavaScript 引擎中尚不受支持。然而，Babel已经支持转译它——[在 Babel REPL 中尝试本文中的示例](https://babeljs.io/repl/#?code_lz=MYewdgzgLgBATgUxgXhgegNoYIYFoBmAugGTEbC4AWhhaAbgNwBQTaaMAKpQJYQy8xKAVwDmSQCgEMKHACeMIWFABbJQjBRuYEfygBCVmlCRYCJSABW3FOgA6ABwDeAJQDiASQD6AUTOWAvvTMQA\&presets=stage-3)!下面的支持表链接到跟踪问题，您可以订阅更新。

<feature-support chrome="no https://bugs.chromium.org/p/v8/issues/detail?id=11935"
              firefox="no https://bugzilla.mozilla.org/show_bug.cgi?id=regexp-v-flag"
              safari="no https://bugs.webkit.org/show_bug.cgi?id=regexp-v-flag"
              nodejs="no"
              babel="7.17.0 https://babeljs.io/blog/2022/02/02/7.17.0"></feature-support>
