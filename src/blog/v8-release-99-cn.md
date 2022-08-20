***

标题： 'V8 版本 v9.9'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），在他的99%'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2022-01-31
    标签：
*   释放
    描述： “V8 版本 v9.9 带来了新的国际化 API。
    推文：“1488190967727411210”

***

每四周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.9)，直到几周后与Chrome 99 Stable合作发布之前，它一直处于测试阶段。V8 v9.9 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 区域设置范围扩展

在 v7.4 中，我们启动了[`Intl.Locale`应用程序接口](https://v8.dev/blog/v8-release-74#intl.locale).在 v9.9 中，我们向`Intl.Locale`对象：`calendars`,`collations`,`hourCycles`,`numberingSystems`,`timeZones`,`textInfo`和`weekInfo`.

这`calendars`,`collations`,`hourCycles`,`numberingSystems`和`timeZones`的属性`Intl.Locale`返回常用标识符的数组，这些标识符旨在与其他标识符一起使用`Intl`应用程序接口：

```js
const arabicEgyptLocale = new Intl.Locale('ar-EG')
// ar-EG
arabicEgyptLocale.calendars
// ['gregory', 'coptic', 'islamic', 'islamic-civil', 'islamic-tbla']
arabicEgyptLocale.collations
// ['compat', 'emoji', 'eor']
arabicEgyptLocale.hourCycles
// ['h12']
arabicEgyptLocale.numberingSystems
// ['arab']
arabicEgyptLocale.timeZones
// ['Africa/Cairo']
```

这`textInfo`的属性`Intl.Locale`返回一个对象以指定与文本相关的信息。目前它只有一个属性，`direction`，以指示区域设置中文本的默认方向性。它被设计用于[断续器`dir`属性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/dir)和[断续器`direction`财产](https://developer.mozilla.org/en-US/docs/Web/CSS/direction).它表示字符的顺序 -`ltr`（从左到右）或`rtl`（从右到左）：

```js
arabicEgyptLocale.textInfo
// { direction: 'rtl' }
japaneseLocale.textInfo
// { direction: 'ltr' }
chineseTaiwanLocale.textInfo
// { direction: 'ltr' }
```

这`weekInfo`的属性`Intl.Locale`返回一个对象以指定与周相关的信息。这`firstDay`return 对象中的属性是一个数字，范围从 1 到 7，指示出于日历目的，将一周中的哪一天视为第一天。1 指定星期一、2 - 星期二、3 - 星期三、4 - 星期四、5 - 星期五、6 - 星期六和 7 - 星期日。这`minimalDays`返回对象中的属性是出于日历目的，在一个月或一年的第一周所需的最小天数。这`weekend`属性在返回对象中是一个整数数组，通常有两个元素，编码方式与`firstDay`.出于日历目的，它指示一周中的哪几天被视为“周末”的一部分。请注意，每个区域设置中周末的天数不同，并且可能不连续。

```js
arabicEgyptLocale.weekInfo
// {firstDay: 6, weekend: [5, 6], minimalDays: 1}
// First day of the week is Saturday. Weekend is Friday and Saturday.
// The first week of a month or a year is a week which has at least 1
// day in that month or year.
```

### 国际枚举

在 v9.9 中，我们添加了一个新函数[`Intl.supportedValuesOf(code)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/supportedValuesOf)返回 v8 中为 Intl API 支持的标识符数组。支持的`code`值为`calendar`,`collation`,`currency`,`numberingSystem`,`timeZone`和`unit`.此新方法中的信息旨在使 Web 开发人员能够轻松发现实现支持的值。

```js
Intl.supportedValuesOf('calendar')
// ['buddhist', 'chinese', 'coptic', 'dangi', ...]

Intl.supportedValuesOf('collation')
// ['big5han', 'compat', 'dict', 'emoji', ...]

Intl.supportedValuesOf('currency')
// ['ADP', 'AED', 'AFA', 'AFN', 'ALK', 'ALL', 'AMD', ...]

Intl.supportedValuesOf('numberingSystem')
// ['adlm', 'ahom', 'arab', 'arabext', 'bali', ...]

Intl.supportedValuesOf('timeZone')
// ['Africa/Abidjan', 'Africa/Accra', 'Africa/Addis_Ababa', 'Africa/Algiers', ...]

Intl.supportedValuesOf('unit')
// ['acre', 'bit', 'byte', 'celsius', 'centimeter', ...]
```

## V8 接口

请使用`git log branch-heads/9.8..branch-heads/9.9 include/v8\*.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.9 -t branch-heads/9.9`以试验 V8 v9.9 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
