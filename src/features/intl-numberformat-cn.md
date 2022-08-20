***

标题： '`Intl.NumberFormat`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)）和谢恩·卡尔
化身：

*   'mathias-bynens'
*   “Shane-carr”
    日期： 2019-08-08
    标签：
*   国际
*   io19
    描述： 'Intl.NumberFormat 启用区域设置感知数字格式。
    推文：“1159476407329873920”

***

您可能已经熟悉`Intl.NumberFormat`API，因为它已经在现代环境中得到了一段时间的支持。

<feature-support chrome="24"
              firefox="29"
              safari="10"
              nodejs="0.12"
              babel="yes"></feature-support>

在其最基本的形式中，`Intl.NumberFormat`允许您创建支持区域设置感知数字格式的可重用格式化程序实例。就像其他人一样`Intl.*Format`API，格式化程序实例同时支持`format`和`formatToParts`方法：

```js
const formatter = new Intl.NumberFormat('en');
formatter.format(987654.321);
// → '987,654.321'
formatter.formatToParts(987654.321);
// → [
// →   { type: 'integer', value: '987' },
// →   { type: 'group', value: ',' },
// →   { type: 'integer', value: '654' },
// →   { type: 'decimal', value: '.' },
// →   { type: 'fraction', value: '321' }
// → ]
```

**注意：**虽然大部分`Intl.NumberFormat`功能可以使用以下方法实现`Number.prototype.toLocaleString`,`Intl.NumberFormat`通常是更好的选择，因为它允许创建一个可重用的格式化程序实例，该实例往往是[更高效](/blog/v8-release-76#localized-bigint).

最近，`Intl.NumberFormat`API 获得了一些新功能。

## `BigInt`支持

除了`Number`s,`Intl.NumberFormat`现在还可以格式化[`BigInt`s](/features/bigint):

```js
const formatter = new Intl.NumberFormat('fr');
formatter.format(12345678901234567890n);
// → '12 345 678 901 234 567 890'
formatter.formatToParts(123456n);
// → [
// →   { type: 'integer', value: '123' },
// →   { type: 'group', value: ' ' },
// →   { type: 'integer', value: '456' }
// → ]
```

<feature-support chrome="76 /blog/v8-release-76#localized-bigint"
              firefox="no"
              safari="no"
              nodejs="no"
              babel="no"></feature-support>

## 测量单位 { #units }

`Intl.NumberFormat`目前支持以下所谓的*简单单位*:

*   角度：`degree`
*   面积：`acre`,`hectare`
*   浓度：`percent`
*   数字：`bit`,`byte`,`kilobit`,`kilobyte`,`megabit`,`megabyte`,`gigabit`,`gigabyte`,`terabit`,`terabyte`,`petabyte`
*   期间：`millisecond`,`second`,`minute`,`hour`,`day`,`week`,`month`,`year`
*   长度：`millimeter`,`centimeter`,`meter`,`kilometer`,`inch`,`foot`,`yard`,`mile`,`mile-scandinavian`
*   质量：`gram`,`kilogram`,`ounce`,`pound`,`stone`
*   温度：`celsius`,`fahrenheit`
*   卷：`liter`,`milliliter`,`gallon`,`fluid-ounce`

要使用本地化单位设置数字格式，请使用`style`和`unit`选项：

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'kilobyte',
});
formatter.format(1.234);
// → '1.234 kB'
formatter.format(123.4);
// → '123.4 kB'
```

请注意，随着时间的推移，可能会添加更多单位的支持。请参阅规格[最新列表](https://tc39.es/proposal-unified-intl-numberformat/section6/locales-currencies-tz_proposed_out.html#table-sanctioned-simple-unit-identifiers).

上述简单单位可以组合成任意分子和分母对，以表示复合单位，例如“升/英亩”或“米每秒”：

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'meter-per-second',
});
formatter.format(299792458);
// → '299,792,458 m/s'
```

<feature-support chrome="77"
              firefox="no"
              safari="no"
              nodejs="no"
              babel="no"></feature-support>

## 紧凑、科学和工程符号 { #notation }

*紧凑表示法*使用特定于区域设置的符号来表示大数字。它是科学记数法的更人性化的替代方案：

```js
{
  // Test standard notation.
  const formatter = new Intl.NumberFormat('en', {
    notation: 'standard', // This is the implied default.
  });
  formatter.format(1234.56);
  // → '1,234.56'
  formatter.format(123456);
  // → '123,456'
  formatter.format(123456789);
  // → '123,456,789'
}

{
  // Test compact notation.
  const formatter = new Intl.NumberFormat('en', {
    notation: 'compact',
  });
  formatter.format(1234.56);
  // → '1.2K'
  formatter.format(123456);
  // → '123K'
  formatter.format(123456789);
  // → '123M'
}
```

：：：备注
**注意：**默认情况下，紧凑表示法舍入为最接近的整数，但始终保留 2 位有效数字。您可以设置`{minimum,maximum}FractionDigits`或`{minimum,maximum}SignificantDigits`以覆盖该行为。
:::

`Intl.NumberFormat`还可以格式化数字[科学记数法](https://en.wikipedia.org/wiki/Scientific_notation):

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'meter-per-second',
  notation: 'scientific',
});
formatter.format(299792458);
// → '2.998E8 m/s'
```

[工程符号](https://en.wikipedia.org/wiki/Engineering_notation)也受支持：

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'meter-per-second',
  notation: 'engineering',
});
formatter.format(299792458);
// → '299.792E6 m/s'
```

<feature-support chrome="77"
              firefox="no"
              safari="no"
              nodejs="no"
              babel="no"></feature-support>

## 标志显示 { #sign }

在某些情况下（例如表示增量），即使数字为正数，显式显示符号也会有所帮助。新`signDisplay`选项启用此功能：

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'percent',
  signDisplay: 'always',
});
formatter.format(-12.34);
// → '-12.34%'
formatter.format(12.34);
// → '+12.34%'
formatter.format(0);
// → '+0%'
formatter.format(-0);
// → '-0%'
```

防止在值为`0`用`signDisplay: 'exceptZero'`:

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'percent',
  signDisplay: 'exceptZero',
});
formatter.format(-12.34);
// → '-12.34%'
formatter.format(12.34);
// → '+12.34%'
formatter.format(0);
// → '0%'
// Note: -0 still displays with a sign, as you’d expect:
formatter.format(-0);
// → '-0%'
```

对于货币，`currencySign`选项启用*会计格式*，从而为负货币金额启用特定于区域设置的格式;例如，将金额括在括号中：

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'currency',
  currency: 'USD',
  signDisplay: 'exceptZero',
  currencySign: 'accounting',
});
formatter.format(-12.34);
// → '($12.34)'
formatter.format(12.34);
// → '+$12.34'
formatter.format(0);
// → '$0.00'
formatter.format(-0);
// → '($0.00)'
```

<feature-support chrome="77"
              firefox="no"
              safari="no"
              nodejs="no"
              babel="no"></feature-support>

## 更多信息

相关[规格建议](https://github.com/tc39/proposal-unified-intl-numberformat)包含更多信息和示例，包括有关如何对每个个体进行特征检测的指导`Intl.NumberFormat`特征。
