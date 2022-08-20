***

标题： '`Intl.DisplayNames`'
作者： '郭淑宇 （[@\_shu](https://twitter.com/\_shu)）和弗兰克·唐
化身：

*   “舒玉国”
*   “坦唐”
    日期： 2020-02-13
    标签：
*   国际
*   节点.js 14
    描述：“Intl.DisplayNames API 支持语言、地区、脚本和货币的本地化名称。
    推文：“1232333889005334529”

***

达到全球受众的 Web 应用程序需要以许多不同的语言显示语言、区域、脚本和货币的显示名称。这些名称的翻译需要数据，这些数据可在[Unicode CLDR](http://cldr.unicode.org/translation/).将数据打包为应用程序的一部分会占用开发人员的时间。用户可能更喜欢语言和地区名称的一致翻译，并且需要不断维护这些数据与世界地缘政治事件保持同步。

幸运的是，大多数JavaScript运行时已经发布并保持最新的翻译数据。新`Intl.DisplayNames`API使JavaScript开发人员能够直接访问这些翻译，从而使应用程序能够更轻松地显示本地化名称。

## 使用示例

下面的示例演示如何创建`Intl.DisplayNames`对象以获取英文区域名称[ISO-3166 2 个字母的国家/地区代码](https://www.iso.org/iso-3166-country-codes.html).

```js
const regionNames = new Intl.DisplayNames(['en'], { type: 'region' });
regionNames.of('US');
// → 'United States'
regionNames.of('BA');
// → 'Bosnia & Herzegovina'
regionNames.of('MM');
// → 'Myanmar (Burma)'
```

以下示例使用繁体中文获取语言名称[Unicode 的语言标识符语法](http://unicode.org/reports/tr35/#Unicode_language_identifier).

```js
const languageNames = new Intl.DisplayNames(['zh-Hant'], { type: 'language' });
languageNames.of('fr');
// → '法文'
languageNames.of('zh');
// → '中文'
languageNames.of('de');
// → '德文'
```

以下示例使用简体中文获取货币名称[ISO-4217 3 字母货币代码](https://www.iso.org/iso-4217-currency-codes.html).在具有不同单数和复数形式的语言中，货币名称是单数。对于复数形式，[`Intl.NumberFormat`](https://v8.dev/features/intl-numberformat)可以使用。

```js
const currencyNames = new Intl.DisplayNames(['zh-Hans'], {type: 'currency'});
currencyNames.of('USD');
// → '美元'
currencyNames.of('EUR');
// → '欧元'
currencyNames.of('JPY');
// → '日元'
currencyNames.of('CNY');
// → '人民币'
```

以下示例显示了最终支持的字体，即英文脚本，使用[ISO-15924 4 个字母的脚本代码](http://unicode.org/iso15924/iso15924-codes.html).

```js
const scriptNames = new Intl.DisplayNames(['en'], { type: 'script' });
scriptNames.of('Latn');
// → 'Latin'
scriptNames.of('Arab');
// → 'Arabic'
scriptNames.of('Kana');
// → 'Katakana'
```

对于更高级的用法，第二个`options`参数还支持`style`财产。这`style`属性对应于显示名称的宽度，可以是`"long"`,`"short"`或`"narrow"`.不同样式的值并不总是不同的。默认值为`"long"`.

```js
const longLanguageNames = new Intl.DisplayNames(['en'], { type: 'language' });
longLanguageNames.of('en-US');
// → 'American English'
const shortLanguageNames = new Intl.DisplayNames(['en'], { type: 'language', style: 'short' });
shortLanguageNames.of('en-US');
// → 'US English'
const narrowLanguageNames = new Intl.DisplayNames(['en'], { type: 'language', style: 'narrow' });
narrowLanguageNames.of('en-US');
// → 'US English'
```

## 完整的接口

完整的 API`Intl.DisplayNames`如下所示。

```js
Intl.DisplayNames(locales, options)
Intl.DisplayNames.prototype.of( code )
```

构造函数与其他构造函数一致`Intl`蜜蜂属。它的第一个参数是[区域设置列表](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl#Locale_identification_and_negotiation)，并且其第二个参数是`options`参数，采用`localeMatcher`,`type`和`style`性能。

这`"localeMatcher"`属性的处理方式与[其他`Intl`蜜蜂属](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl#Locale_identification_and_negotiation).这`type`属性可能是`"region"`,`"language"`,`"currency"`或`"script"`.这`style`属性可能是`"long"`,`"short"`或`"narrow"`跟`"long"`是默认值。

`Intl.DisplayNames.prototype.of( code )`需要以下格式，具体取决于`type`实例的构造方式。

*   什么时候`type`是`"region"`,`code`必须是[ISO-3166 2 个字母的国家/地区代码](https://www.iso.org/iso-3166-country-codes.html)阿尔纳[UN M49 3 位区域代码](https://unstats.un.org/unsd/methodology/m49/).
*   什么时候`type`是`"language"`,`code`必须符合[Unicode 的语言标识符语法](https://unicode.org/reports/tr35/#Unicode_language_identifier).
*   什么时候`type`是`"currency"`,`code`必须是[ISO-4217 3 字母货币代码](https://www.iso.org/iso-4217-currency-codes.html).
*   什么时候`type`是`"script"`,`code`必须是[ISO-15924 4 个字母的脚本代码](https://unicode.org/iso15924/iso15924-codes.html).

## 结论

像其他`Intl`API，作为`Intl.DisplayNames`随着可用性的提高，库和应用程序将选择放弃打包并运送自己的翻译数据，以支持使用本机功能。

## `Intl.DisplayNames`支持 { #support }

<feature-support chrome="81 /blog/v8-release-81#intl.displaynames"
              firefox="86 https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/86#javascript"
              safari="14 https://bugs.webkit.org/show_bug.cgi?id=209779"
              nodejs="14 https://medium.com/@nodejs/node-js-version-14-available-now-8170d384567e"
              babel="no"></feature-support>
