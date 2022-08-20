***

标题： 'V8 版本 v7.9'
作者：“圣地亚哥·阿博伊·索兰内斯，指针式压缩机非凡者”
化身：

*   “圣地亚哥-阿博伊-索兰内斯”
    日期： 2019-11-20
    标签：
*   释放
    描述： 'V8 v7.9 功能删除了对双⇒标记转换的弃用，处理内置中的 API getters，OSR 缓存以及对多个代码空间的 Wasm 支持。
    推文：“1197187184304050176”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.9)，直到几周后与Chrome 79 Stable合作发布之前，它一直处于测试阶段。V8 v7.9 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 性能（大小和速度）{ #performance }

### 删除了双⇒标记过渡的弃用

您可能还记得，在之前的博客文章中，V8 跟踪字段在对象形状中的表示方式。当字段的表示形式发生更改时，必须“弃用”当前对象的形状，并使用新的字段表示形式创建新形状。

一个例外是当旧字段值保证与新表示形式兼容时。在这些情况下，我们可以简单地在对象形状上就地交换新的表示形式，它仍然适用于旧对象的字段值。在 V8 v7.6 中，我们为 Smi ⇒ Tagged 和 HeapObject ⇒ Tagged 转换启用了这些就地表示更改，但由于我们的 MutableHeapNumber 优化，我们无法避免 Double ⇒ Tagged。

在 V8 v7.9 中，我们摆脱了可变HeapNumber，而是使用当它们属于 Double 表示字段时隐式可变的 HeapNumber。这意味着我们必须在处理HeapNumbers时更加小心（如果它们位于双字段上，它们现在是可变的，否则是不可变的），但是HeapNumbers与Taged表示形式兼容，因此我们也可以避免在Double ⇒ Tagged情况下弃用。

这个相对简单的变化将速度计AngularJS分数提高了4%。

![Speedometer AngularJS score improvements](/\_img/v8-release-79/speedometer-angularjs.svg)

### 在内置中处理 API 获取器

以前，V8 在处理由嵌入 API 定义的 getter（如 Blink）时，总是会错过C++运行时。这些包括在HTML规范中定义的getter，例如`Node.nodeType`,`Node.nodeName`等。

V8 将在内置中执行整个原型演练以加载 getter，然后在它意识到 getter 由 API 定义时拯救到运行时。在C++运行时，它会在执行之前遍历原型链以再次获取获取器，从而重复大量工作。

通常[内联缓存 （IC） 机制](https://mathiasbynens.be/notes/shapes-ics)可以帮助缓解这种情况，因为 V8 会在C++运行时第一次错过后安装 IC 处理程序。但是有了新的[惰性反馈分配](https://v8.dev/blog/v8-release-77#lazy-feedback-allocation)，V8 在执行函数一段时间后才会安装 IC 处理程序。

现在，在 V8 v7.9 中，这些 getter 在内置中处理，即使它们没有安装 IC 处理程序，也不必错过C++运行时，通过利用可以直接调用 API getter 的特殊 API 存根。这导致速度计的Backbone和jQuery基准测试中IC运行时间缩短了12%。

![Speedometer Backbone and jQuery improvements](/\_img/v8-release-79/speedometer.svg)

### OSR 缓存

当 V8 确定某些函数是热的时，它会将它们标记为在下一次调用时进行优化。当函数再次执行时，V8 使用优化编译器编译函数，并开始使用后续调用中的优化代码。但是，对于具有长时间运行循环的函数，这是不够的。V8 使用一种称为堆栈上替换 （OSR） 的技术为当前正在执行的函数安装优化代码。这允许我们在函数的第一次执行期间开始使用优化的代码，而它被困在热循环中。

如果该函数第二次执行，则很可能再次被OSRed。在 V8 v7.9 之前，我们需要再次重新优化该函数，以便对其进行 OSR。但是，从 v7.9 开始，我们添加了 OSR 缓存以保留用于 OSR 替换的优化代码，该代码由用作 OSRed 函数入口点的循环标头键入。这已将某些峰值性能基准测试的性能提高了 5–18%。

![OSR caching improvements](/\_img/v8-release-79/osr-caching.svg)

## WebAssembly

### 支持多个代码空间

到目前为止，每个WebAssembly模块只包含一个64位架构上的代码空间，这是在模块创建时保留的。这允许我们在模块中使用近调用，但将 arm64 上的代码空间限制为 128 MB，并且需要在 x64 上预先保留 1 GB。

在 v7.9 中，V8 在 64 位架构上获得了对多个代码空间的支持。这允许我们仅保留估计所需的代码空间，并在以后需要时添加更多代码空间。远跳用于代码空间之间的调用，这些代码空间相距太远，无法进行近距离跳转。现在，V8 支持数百万个 WebAssembly 模块，而不是每个进程约 1000 个，仅受实际可用内存量的限制。

## V8 接口

请使用`git log branch-heads/7.8..branch-heads/7.9 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.9 -t branch-heads/7.9`以试验 V8 v7.9 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
