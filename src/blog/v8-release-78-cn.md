***

标题： 'V8 版本 v7.8'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），懒惰的源码器
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2019-09-27
    标签：
*   释放
    描述： 'V8 v7.8 具有预加载上的流式编译、WebAssembly C API、更快的对象析构和 RegExp 匹配，以及改进的启动时间。
    推文：“1177600702861971459”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.8](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.8)，直到几周后与Chrome 78 Stable合作发布之前，它一直处于测试阶段。V8 v7.8 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript performance （size & speed） { #performance }

### 预加载时的脚本流式处理

你可能还记得[我们从 V8 v7.5 开始的脚本流工作](/blog/v8-release-75#script-streaming-directly-from-network)，我们改进了后台编译，以直接从网络读取数据。在 Chrome 78 中，我们在预加载期间启用脚本流式传输。

以前，脚本流在`<script>`标记，在 HTML 解析过程中遇到，解析将暂停直到编译完成（对于普通脚本），或者脚本将在完成编译后执行（对于异步脚本）。这意味着对于像这样的普通同步脚本：

```html
<!DOCTYPE html>
<html>
<head>
  <script src="main.js"></script>
</head>
...
```

...管道以前大致如下所示：

<figure>
  <img src="/_img/v8-release-78/script-streaming-0.svg" width="458" height="130" alt="" loading="lazy">
</figure>

由于同步脚本可以使用`document.write()`，我们必须暂停解析HTML，当我们看到`<script>`标记。由于编译开始时`<script>`标记，解析 HTML 和实际运行脚本之间存在很大差距，在此期间我们无法继续加载页面。

但是，我们*也*遇到`<script>`标记中，我们扫描 HTML 以查找要预加载的资源，因此管道实际上更像这样：

<figure>
  <img src="/_img/v8-release-78/script-streaming-1.svg" width="600" height="130" alt="" loading="lazy">
</figure>

这是一个相当安全的假设，如果我们预加载一个JavaScript文件，我们最终会想要执行它。因此，自Chrome 76以来，我们一直在尝试预加载流，其中加载脚本也开始编译它。

<figure>
  <img src="/_img/v8-release-78/script-streaming-2.svg" width="495" height="130" alt="" loading="lazy">
</figure>

更好的是，由于我们可以在脚本完成加载之前开始编译，因此具有预加载流的管道实际上看起来更像这样：

<figure>
  <img src="/_img/v8-release-78/script-streaming-3.svg" width="480" height="217" alt="" loading="lazy">
</figure>

这意味着在某些情况下，我们可以减少可感知的编译时间（`<script>`-标记已查看和脚本开始执行）降至零。在我们的实验中，这种可感知的编译时间平均下降了5-20%。

最好的消息是，由于我们的实验基础设施，我们不仅能够在Chrome 78中默认启用此功能，还可以为Chrome 76以后的用户启用它。

### 更快的对象解构

对象解构形式...

```js
const {x, y} = object;
```

...几乎等同于脱糖形式...

```js
const x = object.x;
const y = object.y;
```

...除了它还需要抛出一个特殊的错误`object`存在`undefined`或`null`...

    $ v8 -e 'const object = undefined; const {x, y} = object;'
    unnamed:1: TypeError: Cannot destructure property `x` of 'undefined' or 'null'.
    const object = undefined; const {x, y} = object;
                                     ^

...而不是尝试取消引用未定义时得到的正常错误：

    $ v8 -e 'const object = undefined; object.x'
    unnamed:1: TypeError: Cannot read property 'x' of undefined
    const object = undefined; object.x
                                     ^

这个额外的检查使得解构比简单的变量赋值慢，因为[通过推特向我们举报](https://twitter.com/mkubilayk/status/1166360933087752197).

从 V8 v7.8 开始，对象析构是**一样快**作为等效的去糖化变量赋值（实际上，我们为两者生成相同的字节码）。现在，而不是显式`undefined`/`null`检查，我们依赖于加载时引发的异常`object.x`，如果异常是解构的结果，我们会截获该异常。

### 懒惰的源位置

从 JavaScript 编译字节码时，会生成源位置表，将字节码序列与源代码中的字符位置相关联。但是，此信息仅在符号化异常或执行开发人员任务（如调试和分析）时使用，因此这在很大程度上会浪费内存。

为了避免这种情况，我们现在在不收集源位置的情况下编译字节码（假设没有附加调试器或探查器）。仅当实际生成堆栈跟踪时（例如，在调用时），才会收集源位置`Error.stack`或将异常的堆栈跟踪打印到控制台。这确实有一些成本，因为生成源位置需要重新分析和编译函数，但是大多数网站不会在生产中对堆栈跟踪进行符号化，因此看不到任何可观察到的性能影响。在我们的实验室测试中，我们看到 V8 的内存使用量减少了 1-2.5%。

![Memory savings from lazy source positions on an AndroidGo device](/\_img/v8-release-78/memory-savings.svg)

### 更快的正则表达式匹配失败

通常，RegExp 尝试通过向前循环访问输入字符串并检查从每个位置开始的匹配项来查找匹配项。一旦该位置足够接近字符串的末尾，以至于不可能进行匹配，V8 现在（在大多数情况下）停止尝试查找新匹配的可能开始，而是快速返回失败。此优化适用于已编译和已解释的正则表达式，并在无法找到匹配项的常见工作负载上产生加速，并且与平均输入字符串长度相比，任何成功匹配的最小长度相对较大。

在 JetStream 2 中的 UniPoker 测试中，V8 v7.8 将平均迭代子分数提高了 20%。

## WebAssembly

### WebAssembly C/C++ API

从 v7.8 开始，V8 的实现[Wasm C/C++ API](https://github.com/WebAssembly/wasm-c-api)从实验状态毕业到得到官方支持。它允许您在 C/C++ 应用程序中使用 V8 的特殊版本作为 WebAssembly 执行引擎。不涉及 JavaScript！有关更多详细信息和说明，请参阅[文档](https://docs.google.com/document/d/1oFPHyNb_eXg6NzrE6xJDNPdJrHMZvx0LqsD6wpbd9vY/edit).

### 缩短了启动时间

从 WebAssembly 调用 JavaScript 函数或从 JavaScript 调用 WebAssembly 函数涉及执行一些包装器代码，负责将函数的参数从一种表示形式转换为另一种表示形式。 生成这些包装器可能非常昂贵：在[史诗禅园演示](https://s3.amazonaws.com/mozilla-games/ZenGarden/EpicZenGarden.html)，编译包装器在 18 核 Xeon 计算机上大约需要 20% 的模块启动时间（编译 + 实例化）。

对于此版本，我们通过更好地利用多核计算机上的后台线程来改善这种情况。我们依靠最近的努力[缩放函数编译](/blog/v8-release-77#wasm-compilation)，并将包装器编译集成到这个新的异步管道中。包装器编译现在占同一台机器上Epic ZenGarden演示启动时间的8%左右。

## V8 接口

请使用`git log branch-heads/7.7..branch-heads/7.8 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.8 -t branch-heads/7.8`以试验 V8 v7.8 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
