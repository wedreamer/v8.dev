***

标题： 'V8 发布 v5.7'
作者： 'V8团队'
日期： 2017-02-06 13：33：37
标签：

*   释放
    描述： 'V8 v5.7 默认启用 WebAssembly，包括性能改进和对 ECMAScript 语言功能的增强支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.7)，它将处于测试阶段，直到几周后与Chrome 57 Stable协同发布。V8 5.7 充满了各种面向开发人员的好东西。我们想在发布之前预览一些亮点。

## 性能改进

### 本机异步功能与承诺一样快

异步函数现在的速度大约与使用 promise 编写的相同代码一样快。异步函数的执行性能翻了两番，根据我们的[微模板标记](https://codereview.chromium.org/2577393002).同期，整体承诺业绩也翻了一番。

![Async performance improvements in V8 on Linux x64](/\_img/v8-release-57/async.png)

### ES2015 的持续改进

V8 继续使 ES2015 语言功能更快，以便开发人员在不产生性能成本的情况下使用新功能。现在，扩散运算符、解构和生成器[大约和他们幼稚的ES5等效物一样快](https://fhinkel.github.io/six-speed/).

### 正则表达式速度提高 15%

将RegExp函数从自托管JavaScript实现迁移到与TurboFan的代码生成架构挂钩的实现，使RegExp的整体性能提高了约15%。更多细节可以在[专门的博客文章](/blog/speeding-up-regular-expressions).

## JavaScript 语言特性

此版本中包括了 ECMAScript 标准库的几个最新添加内容。两个字符串方法，[`padStart`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/padStart)和[`padEnd`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/padEnd)，提供了有用的字符串格式设置功能，而[`Intl.DateTimeFormat.prototype.formatToParts`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DateTimeFormat/formatToParts)使作者能够以区域设置感知的方式自定义其日期/时间格式。

## WebAssembly enabled

Chrome 57（包括 V8 v5.7）将是第一个默认启用 WebAssembly 的版本。有关更多详细信息，请参阅[webassembly.org](http://webassembly.org/)和 API 文档[MDN](https://developer.mozilla.org/en-US/docs/WebAssembly/API).

## V8 接口新增功能

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.7 -t branch-heads/5.7`以试验 V8 v5.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。

### `PromiseHook`

此C++ API 允许用户实现跟踪承诺生命周期的分析代码。这使得Node即将推出[AsyncHook API](https://github.com/nodejs/node-eps/pull/18)它允许您构建[异步上下文传播](https://docs.google.com/document/d/1tlQ0R6wQFGqCS5KeIw0ddoLbaSYx6aU7vyXOkv-wvlM/edit#).

这`PromiseHook`API 提供了四个生命周期挂钩：初始化、解析、之前和之后。init 钩子在创建新承诺时运行;解析钩子在解析承诺时运行;前后钩子在[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob).有关更多信息，请查看[跟踪问题](https://bugs.chromium.org/p/v8/issues/detail?id=4643)和[设计文档](https://docs.google.com/document/d/1rda3yKGHimKIhg5YeoAmCOtyURgsbTH_qaYR79FELlk/edit).
