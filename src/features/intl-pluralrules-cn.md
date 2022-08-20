***

标题： '`Intl.PluralRules`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2017-10-04
    标签：
*   国际
    描述： '处理复数是许多看起来很简单的问题之一，直到你意识到每种语言都有自己的复数规则。International.PluralRules API 可以提供帮助！
    推文：“915542989493202944”

***

Iñtërnâtiônàlizætiøn is hard.处理复数是许多看起来很简单的问题之一，直到你意识到每种语言都有自己的复数规则。

对于英语复数化，只有两种可能的结果。让我们以“猫”这个词为例：

*   1 只猫，即`'one'`形式，在英语中称为单数
*   2只猫，还有42只猫，0.5只猫，等等，即`'other'`形式（唯一的其他），在英语中称为复数。

全新[`Intl.PluralRules`应用程序接口](https://github.com/tc39/proposal-intl-plural-rules)告诉您哪种形式适用于基于给定数字选择的语言。

```js
const pr = new Intl.PluralRules('en-US');
pr.select(0);   // 'other' (e.g. '0 cats')
pr.select(0.5); // 'other' (e.g. '0.5 cats')
pr.select(1);   // 'one'   (e.g. '1 cat')
pr.select(1.5); // 'other' (e.g. '0.5 cats')
pr.select(2);   // 'other' (e.g. '0.5 cats')
```

与其他国际化 API 不同，`Intl.PluralRules`是一个低级 API，它本身不执行任何格式化。相反，您可以在其上构建自己的格式化程序：

```js
const suffixes = new Map([
  // Note: in real-world scenarios, you wouldn’t hardcode the plurals
  // like this; they’d be part of your translation files.
  ['one',   'cat'],
  ['other', 'cats'],
]);
const pr = new Intl.PluralRules('en-US');
const formatCats = (n) => {
  const rule = pr.select(n);
  const suffix = suffixes.get(rule);
  return `${n} ${suffix}`;
};

formatCats(1);   // '1 cat'
formatCats(0);   // '0 cats'
formatCats(0.5); // '0.5 cats'
formatCats(1.5); // '1.5 cats'
formatCats(2);   // '2 cats'
```

对于相对简单的英语复数规则，这似乎有些过分;但是，并非所有语言都遵循相同的规则。有些语言只有一种复数形式，有些语言有多种形式。[威尔士语](http://unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html#rules)，例如，有六种不同的复数形式！

```js
const suffixes = new Map([
  ['zero',  'cathod'],
  ['one',   'gath'],
  // Note: the `two` form happens to be the same as the `'one'`
  // form for this word specifically, but that is not true for
  // all words in Welsh.
  ['two',   'gath'],
  ['few',   'cath'],
  ['many',  'chath'],
  ['other', 'cath'],
]);
const pr = new Intl.PluralRules('cy');
const formatWelshCats = (n) => {
  const rule = pr.select(n);
  const suffix = suffixes.get(rule);
  return `${n} ${suffix}`;
};

formatWelshCats(0);   // '0 cathod'
formatWelshCats(1);   // '1 gath'
formatWelshCats(1.5); // '1.5 cath'
formatWelshCats(2);   // '2 gath'
formatWelshCats(3);   // '3 cath'
formatWelshCats(6);   // '6 chath'
formatWelshCats(42);  // '42 cath'
```

为了在支持多种语言的同时实现正确的复数化，需要一个语言数据库及其复数规则。[The Unicode CLDR](http://cldr.unicode.org/)包含此数据，但要在 JavaScript 中使用它，它必须与其他 JavaScript 代码一起嵌入和发布，从而增加加载时间、解析时间和内存使用量。这`Intl.PluralRules`API将这一负担转移到JavaScript引擎上，从而实现了更高性能的国际化复数化。

：：：备注
**注意：**虽然 CLDR 数据包括每种语言的表单映射，但它没有附带单个单词的单数/复数形式的列表。你仍然需要自己翻译和提供这些，就像以前一样。
:::

## 序数

这`Intl.PluralRules`API 支持各种选择规则`type`属性`options`论点。其隐式默认值（如上述示例中使用的）为`'cardinal'`.找出给定数字的序数指示符（例如`1`→`1st`,`2`→`2nd`等），用途`{ type: 'ordinal' }`:

```js
const pr = new Intl.PluralRules('en-US', {
  type: 'ordinal'
});
const suffixes = new Map([
  ['one',   'st'],
  ['two',   'nd'],
  ['few',   'rd'],
  ['other', 'th'],
]);
const formatOrdinals = (n) => {
  const rule = pr.select(n);
  const suffix = suffixes.get(rule);
  return `${n}${suffix}`;
};

formatOrdinals(0);   // '0th'
formatOrdinals(1);   // '1st'
formatOrdinals(2);   // '2nd'
formatOrdinals(3);   // '3rd'
formatOrdinals(4);   // '4th'
formatOrdinals(11);  // '11th'
formatOrdinals(21);  // '21st'
formatOrdinals(42);  // '42nd'
formatOrdinals(103); // '103rd'
```

`Intl.PluralRules`是一个低级 API，尤其是与其他国际化功能相比。因此，即使您不直接使用它，也可能使用依赖于它的库或框架。

随着此 API 变得越来越广泛，您会发现诸如[全球化](https://github.com/globalizejs/globalize#plural-module)放弃对硬编码 CLDR 数据库的依赖性，转而使用本机功能，从而提高加载时性能、分析时性能、运行时性能和内存使用率。

## `Intl.PluralRules`支持 { #support }

<feature-support chrome="63 /blog/v8-release-63"
              firefox="58"
              safari="13"
              nodejs="10"
              babel="no"></feature-support>
