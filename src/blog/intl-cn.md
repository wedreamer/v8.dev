***

标题：“更快、功能更丰富的国际化 API”
作者： '[சத்யா குணசேகரன் （Sathya Gunasekaran）](https://twitter.com../_gsathya)'
日期： 2019-04-25 16：45：37
化身：

*   'sathya-gunasekaran'
    标签：
*   ECMAScript
*   国际
    描述： “JavaScript Internationalization API 正在增长，其 V8 实现速度越来越快！
    推文：“1121424877142122500”

***

[The ECMAScript Internationalization API Specification](https://tc39.es/ecma402/)（ECMA-402，或`Intl`） 提供特定于区域设置的关键功能，如日期格式设置、数字格式设置、复数形式选择和排序规则。Chrome V8和Google Internationalization团队一直在合作为V8的ECMA-402实现添加功能，同时清理技术债务并提高性能以及与其他浏览器的互操作性。

## 基础体系结构改进

最初，ECMA-402 规范主要使用 V8 扩展在 JavaScript 中实现，并且位于 V8 代码库之外。使用外部扩展 API 意味着不能使用 V8 的几个内部使用的 API 进行类型检查、外部C++对象的生存期管理和内部私有数据存储。作为提高启动性能的一部分，此实现后来被移至 V8 代码库中，以启用[快照](/blog/custom-startup-snapshots)这些内置的。

V8 使用专用`JSObject`s 与自定义[形状（隐藏类）](https://mathiasbynens.be/notes/shapes-ics)描述由 ECMAScript 指定的内置 JavaScript 对象（如`Promise`s,`Map`s,`Set`等）。通过这种方法，V8 可以预先分配所需数量的内部插槽并生成对这些插槽的快速访问，而不是一次一个属性地增加对象，从而导致性能降低和内存使用率下降。

这`Intl`由于历史性的分裂，实现不是以这样的架构为模型的。相反，所有内置的 JavaScript 对象都由国际化规范指定（如`NumberFormat`,`DateTimeFormat`） 是通用的`JSObject`s，必须通过其内部插槽的多个属性添加进行转换。

另一个没有专门的工件`JSObject`s是类型检查现在更复杂了。类型信息存储在私有符号下，并使用昂贵的属性访问在JS和C++端进行类型检查，而不仅仅是查找其形状。

### 实现代码库现代化

随着当前在 V8 中不再编写自托管内置组件，利用这个机会对 ECMA402 实现进行现代化改造是有意义的。

### 远离自托管 JS

尽管自承载有助于编写简洁易读的代码，但频繁使用缓慢的运行时调用来访问 ICU API 会导致性能问题。因此，许多 ICU 功能在 JavaScript 中被复制，以减少此类运行时调用的数量。

通过重写C++中的内置组件，访问ICU API的速度要快得多，因为现在没有运行时调用开销。

### 改善重症监护室

ICU 是一组 C/C++库，由大量应用程序（包括所有主要的 JavaScript 引擎）使用，用于提供 Unicode 和全球化支持。作为切换的一部分`Intl`到 ICU 在 V8 的实施中，我们[发现](https://unicode-org.atlassian.net/browse/ICU-20140) [和](https://unicode-org.atlassian.net/browse/ICU-9562) [固定](https://unicode-org.atlassian.net/browse/ICU-20098)几个 ICU 错误。

作为实施新提案的一部分，例如[`Intl.RelativeTimeFormat`](/features/intl-relativetimeformat),[`Intl.ListFormat`](/features/intl-listformat)和`Intl.Locale`，我们通过添加[几个](https://unicode-org.atlassian.net/browse/ICU-13256) [新增功能](https://unicode-org.atlassian.net/browse/ICU-20121) [蜜蜂属](https://unicode-org.atlassian.net/browse/ICU-20342)以支持这些新的 ECMAScript 提案。

所有这些新增功能都有助于其他 JavaScript 引擎更快地实现这些建议，从而推动 Web 向前发展！例如，Firefox的开发正在进行中，以实现几个新的`Intl`基于我们ICU工作的API。

## 性能

作为这项工作的结果，我们通过优化多个快速路径并缓存各种路径的初始化来提高国际化 API 的性能`Intl`对象和`toLocaleString`方法`Number.prototype`,`Date.prototype`和`String.prototype`.

例如，创建一个新的`Intl.NumberFormat`物体变得大约24×快。

![Microbenchmarks testing the performance of creating various Intl objects](../_img/intl/performance.svg)

请注意，为了获得更好的性能，建议显式创建*和重用*一`Intl.NumberFormat`或`Intl.DateTimeFormat`或`Intl.Collator`对象，而不是调用类似`toLocaleString`或`localeCompare`.

## 新增功能`Intl`特征

所有这些工作都为构建新功能奠定了良好的基础，我们将继续发布第 3 阶段中的所有新国际化提案。

[`Intl.RelativeTimeFormat`](/features/intl-relativetimeformat)已在 Chrome 71 中发布，[`Intl.ListFormat`](/features/intl-listformat)已在 Chrome 72 中发货，[`Intl.Locale`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Locale)已在 Chrome 74 中发布，并且[`dateStyle`和`timeStyle`选项`Intl.DateTimeFormat`](https://github.com/tc39/proposal-intl-datetime-style)和[BigInt 支持`Intl.DateTimeFormat`](https://github.com/tc39/ecma402/pull/236)在 Chrome 76 中发货。[`Intl.DateTimeFormat#formatRange`](https://github.com/tc39/proposal-intl-DateTimeFormat-formatRange),[`Intl.Segmenter`](https://github.com/tc39/proposal-intl-segmenter/)和[其他选项`Intl.NumberFormat`](https://github.com/tc39/proposal-unified-intl-numberformat/)目前正在V8中开发，我们希望尽快发布它们！

其中许多新 API 以及管道中其他 API 都是由于我们在标准化新功能以帮助开发人员实现国际化方面所做的工作。[`Intl.DisplayNames`](https://github.com/tc39/proposal-intl-displaynames)是一个阶段 1 建议，允许用户本地化语言、区域或脚本显示名称的显示名称。[`Intl.DateTimeFormat#formatRange`](https://github.com/fabalbon/proposal-intl-DateTimeFormat-formatRange)是一个第 3 阶段建议，它指定了一种以简洁且可识别区域设置的方式设置日期范围的格式的方法。[统一`Intl.NumberFormat`原料药提案](https://github.com/tc39/proposal-unified-intl-numberformat)是改进的第 3 阶段提案`Intl.NumberFormat`通过添加对度量单位、货币和符号显示策略以及科学和紧凑符号表示法的支持。您也可以参与 ECMA-402 的未来，通过以下方式做出贡献：[其 GitHub 存储库](https://github.com/tc39/ecma402).

## 结论

`Intl`为国际化 Web 应用所需的多个操作提供了功能丰富的 API，将繁重的工作留给了浏览器，而无需通过网络传送尽可能多的数据或代码。考虑正确使用这些 API 可以使 UI 在不同的区域设置中更好地工作。由于 Google V8 和 i18n 团队与 TC39 及其 ECMA-402 子组合作完成的工作，您现在可以访问更多功能和更好的性能，并期望随着时间的推移进一步改进。
