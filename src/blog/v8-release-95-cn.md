***

标题： 'V8 版本 v9.5'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-09-21
    标签：
*   释放
    描述： “V8 版本 v9.5 带来了更新的国际化 API 和 WebAssembly 异常处理支持。
    推文：“1440296019623759872”

***

每四周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.5)，直到几周后与Chrome 95 Stable一起发布之前，它一直处于测试阶段。V8 v9.5 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `Intl.DisplayNames`v2

在 v8.1 中，我们启动了[`Intl.DisplayNames`应用程序接口](https://v8.dev/features/intl-displaynames)Chrome 81 中的 API，支持的类型为“语言”、“区域”、“脚本”和“货币”。在 v9.5 中，我们现在添加了两个新的受支持类型：“日历”和“dateTimeField”。它们相应地返回不同日历类型和日期时间字段的显示名称：

```js
const esCalendarNames = new Intl.DisplayNames(['es'], { type: 'calendar' });
const frDateTimeFieldNames = new Intl.DisplayNames(['fr'], { type: 'dateTimeField' });
esCalendarNames.of('roc');  // "calendario de la República de China"
frDateTimeFieldNames.of('month'); // "mois"
```

我们还通过新的 languageDisplay 选项增强了对“语言”类型的支持，该选项可以是“标准”或“方言”（如果未指定，则为默认值）：

```js
const jaDialectLanguageNames = new Intl.DisplayNames(['ja'], { type: 'language' });
const jaStandardLanguageNames = new Intl.DisplayNames(['ja'], { type: 'language' , languageDisplay: 'standard'});
jaDialectLanguageNames.of('en-US')  // "アメリカ英語"
jaDialectLanguageNames.of('en-AU')  // "オーストラリア英語"
jaDialectLanguageNames.of('en-GB')  // "イギリス英語"

jaStandardLanguageNames.of('en-US') // "英語 (アメリカ合衆国)"
jaStandardLanguageNames.of('en-AU') // "英語 (オーストラリア)"
jaStandardLanguageNames.of('en-GB') // "英語 (イギリス)"
```

### 扩展`timeZoneName`选择

`Intl.DateTimeFormat API`在 v9.5 中现在支持四个新值`timeZoneName`选择：

*   “shortGeneric”以短通用非位置格式输出时区的名称，例如“PT”，“ET”，而不指示它是否在夏令时内。
*   “longGeneric”以短通用非位置格式输出时区的名称，例如“太平洋时间”，“山地时间”，而不指示它是否在夏令时之下。
*   “shortOffset”以输出时区的名称，如短的本地化 GMT 格式，如“GMT-8”。
*   “longOffset”以输出时区的名称，如长本地化 GMT 格式，如“GMT-0800”。

## WebAssembly

### 异常处理

V8 现在支持[WebAssembly Exception Handling （Wasm EH） 提案](https://github.com/WebAssembly/exception-handling/blob/master/proposals/exception-handling/Exceptions.md)以便使用兼容的工具链编译模块（例如[Emscripten](https://emscripten.org/docs/porting/exceptions.html)） 可以在 V8 中执行。该提案旨在与以前使用JavaScript的解决方法相比，保持较低的开销。

例如，我们编译了[二进制](https://github.com/WebAssembly/binaryen/)WebAssembly的优化器，具有旧的和新的异常处理实现。

启用异常处理后，代码大小会增加[从旧的基于JavaScript的异常处理的约43%下降到新的Wasm EH功能的9%。](https://github.com/WebAssembly/exception-handling/issues/20#issuecomment-919716209).

当我们跑步时`wasm-opt.wasm -O3`在几个大型测试文件中，Wasm EH的版本与基线相比没有出现任何性能损失，而基于JavaScript的EH版本则花费了大约30%的时间。

但是，Binaryen 很少使用异常检查。在异常繁重的工作负载中，性能差异预计会更大。

## V8 接口

主 v8.h 头文件已拆分为几个部分，可以单独包含。例如`v8-isolate.h`现在包含`v8::Isolate class`.许多声明方法传递的头文件`v8::Local<T>`现在可以导入`v8-forward.h`以获取`v8::Local`和所有 v8 堆对象类型。

请使用`git log branch-heads/9.4..branch-heads/9.5 include/v8\*.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.5 -t branch-heads/9.5`以试验 V8 v9.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
