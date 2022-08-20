***

标题： '`Intl.Locale`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-05-20
    标签：
*   国际
*   节点.js 12
*   io19
    描述： “新的 Intl.Locale API 提供了一种统一的机制来处理语言环境，并且比使用字符串更方便。
    推文：“待办事项”

***

在处理[国际化接口](/features/tags/intl)，通常将表示区域设置 ID 的字符串传递给各种`Intl`构造函数，例如`'en'`英语。[新`Intl.Locale`应用程序接口](https://github.com/tc39/proposal-intl-locale)提供了更强大的机制来处理此类区域设置。

它可以轻松提取特定于区域设置的首选项，例如不仅语言，还提取日历，编号系统，小时周期，区域等。

```js
const locale = new Intl.Locale('es-419-u-hc-h12', {
  calendar: 'gregory'
});
locale.language;
// → 'es'
locale.calendar;
// → 'gregory'
locale.hourCycle;
// → 'h12'
locale.region;
// → '419'
locale.toString();
// → 'es-419-u-ca-gregory-hc-h12'
```

## `Intl.Locale`支持 { #support }

<feature-support chrome="74 /blog/v8-release-74#intl.locale"
              firefox="no"
              safari="no"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="no"></feature-support>
