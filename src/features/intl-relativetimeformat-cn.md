***

标题： '`Intl.RelativeTimeFormat`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-10-22
    标签：
*   国际
*   节点.js 12
*   io19
    描述： 'Intl.RelativeTimeFormat 支持在不牺牲性能的情况下对相对时间进行本地化格式化。
    推文：“1054387117571354624”

***

现代Web应用程序通常使用“昨天”，“42秒前”或“3个月内”之类的短语，而不是完整的日期和时间戳。这样*相对时间格式值*已经变得如此普遍，以至于几个流行的库实现了以本地化方式格式化它们的实用程序函数。（示例包括[瞬间.js](https://momentjs.com/),[全球化](https://github.com/globalizejs/globalize)和[日期-fns](https://date-fns.org/docs/).)

实现本地化的相对时间格式化程序的一个问题是，对于要支持的每种语言，您需要一个习惯单词或短语（如“昨天”或“最后一个季度”）的列表。[The Unicode CLDR](http://cldr.unicode.org/)提供了这些数据，但是要在JavaScript中使用它，它必须嵌入并与其他库代码一起发布。不幸的是，这会增加此类库的捆绑包大小，从而对加载时间、解析/编译成本和内存消耗产生负面影响。

全新`Intl.RelativeTimeFormat`API将这种负担转移到JavaScript引擎上，JavaScript引擎可以发布区域设置数据，并使其直接提供给JavaScript开发人员。`Intl.RelativeTimeFormat`支持在不牺牲性能的情况下对相对时间进行本地化格式化。

## 使用示例

下面的示例演示如何使用英语创建相对时间格式化程序。

```js
const rtf = new Intl.RelativeTimeFormat('en');

rtf.format(3.14, 'second');
// → 'in 3.14 seconds'

rtf.format(-15, 'minute');
// → '15 minutes ago'

rtf.format(8, 'hour');
// → 'in 8 hours'

rtf.format(-2, 'day');
// → '2 days ago'

rtf.format(3, 'week');
// → 'in 3 weeks'

rtf.format(-5, 'month');
// → '5 months ago'

rtf.format(2, 'quarter');
// → 'in 2 quarters'

rtf.format(-42, 'year');
// → '42 years ago'
```

请注意，参数传递给`Intl.RelativeTimeFormat`构造函数可以是字符串保持[A BCP 47 语言标签](https://tools.ietf.org/html/rfc5646)或[此类语言标记的数组](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl#Locale_identification_and_negotiation).

下面是使用其他语言（西班牙语）的示例：

```js
const rtf = new Intl.RelativeTimeFormat('es');

rtf.format(3.14, 'second');
// → 'dentro de 3,14 segundos'

rtf.format(-15, 'minute');
// → 'hace 15 minutos'

rtf.format(8, 'hour');
// → 'dentro de 8 horas'

rtf.format(-2, 'day');
// → 'hace 2 días'

rtf.format(3, 'week');
// → 'dentro de 3 semanas'

rtf.format(-5, 'month');
// → 'hace 5 meses'

rtf.format(2, 'quarter');
// → 'dentro de 2 trimestres'

rtf.format(-42, 'year');
// → 'hace 42 años'
```

此外，`Intl.RelativeTimeFormat`构造函数接受可选`options`参数，它提供对输出的细粒度控制。为了说明灵活性，让我们看一下基于默认设置的更多英文输出：

```js
// Create a relative time formatter for the English language, using the
// default settings (just like before). In this example, the default
// values are explicitly passed in.
const rtf = new Intl.RelativeTimeFormat('en', {
  localeMatcher: 'best fit', // other values: 'lookup'
  style: 'long', // other values: 'short' or 'narrow'
  numeric: 'always', // other values: 'auto'
});

// Now, let’s try some special cases!

rtf.format(-1, 'day');
// → '1 day ago'

rtf.format(0, 'day');
// → 'in 0 days'

rtf.format(1, 'day');
// → 'in 1 day'

rtf.format(-1, 'week');
// → '1 week ago'

rtf.format(0, 'week');
// → 'in 0 weeks'

rtf.format(1, 'week');
// → 'in 1 week'
```

您可能已经注意到，上面的格式化程序生成了字符串`'1 day ago'`而不是`'yesterday'`，和略显尴尬`'in 0 weeks'`而不是`'this week'`.发生这种情况是因为默认情况下，格式化程序在输出中使用数值。

若要更改此行为，请将`numeric`选项`'auto'`（而不是隐含的默认值`'always'`):

```js
// Create a relative time formatter for the English language that does
// not always have to use numeric value in the output.
const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

rtf.format(-1, 'day');
// → 'yesterday'

rtf.format(0, 'day');
// → 'today'

rtf.format(1, 'day');
// → 'tomorrow'

rtf.format(-1, 'week');
// → 'last week'

rtf.format(0, 'week');
// → 'this week'

rtf.format(1, 'week');
// → 'next week'
```

类似于其他`Intl`类`Intl.RelativeTimeFormat`具有`formatToParts`方法除了`format`方法。虽然`format`涵盖了最常见的用例，`formatToParts`如果您需要访问生成的输出的各个部分，则可能会有所帮助：

```js
// Create a relative time formatter for the English language that does
// not always have to use numeric value in the output.
const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

rtf.format(-1, 'day');
// → 'yesterday'

rtf.formatToParts(-1, 'day');
// → [{ type: 'literal', value: 'yesterday' }]

rtf.format(3, 'week');
// → 'in 3 weeks'

rtf.formatToParts(3, 'week');
// → [{ type: 'literal', value: 'in ' },
//    { type: 'integer', value: '3', unit: 'week' },
//    { type: 'literal', value: ' weeks' }]
```

有关其余选项及其行为的详细信息，请参阅[提案存储库中的 API 文档](https://github.com/tc39/proposal-intl-relative-time#api).

## 结论

`Intl.RelativeTimeFormat`在 V8 v7.1 和 Chrome 71 中默认可用。随着此 API 变得越来越广泛，您会发现诸如[瞬间.js](https://momentjs.com/),[全球化](https://github.com/globalizejs/globalize)和[日期-fns](https://date-fns.org/docs/)放弃对硬编码 CLDR 数据库的依赖性，转而使用本机相对时间格式设置功能，从而提高加载时性能、分析和编译时性能、运行时性能和内存使用率。

## `Intl.RelativeTimeFormat`支持 { #support }

<feature-support chrome="71 /blog/v8-release-71#javascript-language-features"
              firefox="65"
              safari="14"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="no"></feature-support>
