***

标题： 'V8 发布 v8.1'
作者：“多米尼克·因富尔，国际化神秘人物”
化身：

*   'dominik-infuehr'
    日期： 2020-02-25
    标签：
*   释放
    描述： “V8 v8.1 功能通过新的 Intl.DisplayNames API 改进了国际化支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.1](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.1)，直到几周后与Chrome 81 Stable一起发布之前，它一直处于测试阶段。V8 v8.1 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `Intl.DisplayNames`

新`Intl.DisplayNames`API 允许程序员轻松显示语言、区域、脚本和货币的翻译名称。

```js
const zhLanguageNames = new Intl.DisplayNames(['zh-Hant'], { type: 'language' });
const enRegionNames = new Intl.DisplayNames(['en'], { type: 'region' });
const itScriptNames = new Intl.DisplayNames(['it'], { type: 'script' });
const deCurrencyNames = new Intl.DisplayNames(['de'], {type: 'currency'});

zhLanguageNames.of('fr');
// → '法文'
enRegionNames.of('US');
// → 'United States'
itScriptNames.of('Latn');
// → 'latino'
deCurrencyNames.of('JPY');
// → 'Japanischer Yen'
```

立即将翻译数据维护的负担转移到运行时！看[我们的功能解释器](https://v8.dev/features/intl-displaynames)有关完整 API 的详细信息和更多示例。

## V8 接口

请使用`git log branch-heads/8.0..branch-heads/8.1 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 8.1 -t branch-heads/8.1`以试验 V8 v8.1 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
