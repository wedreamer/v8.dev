***

标题： 'V8 发布 v8.6'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），键盘模糊器'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2020-09-21
    标签：
*   释放
    描述： 'V8 版本 v8.6 带来了令人敬畏的代码、性能改进和规范性更改。
    推文：“1308062287731789825”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.6)，直到几周后与Chrome 86 Stable合作发布之前，它一直处于测试阶段。V8 v8.6 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 尊重代码

v8.6 版本使 V8 代码库[更尊重](https://v8.dev/docs/respectful-code).该团队加入了Chromium范围的努力，通过替换项目中的一些不敏感的术语来遵循谷歌对种族平等的承诺。这仍然是一项持续的努力，欢迎任何外部贡献者伸出援手！您可以看到仍然可用的任务列表[这里](https://docs.google.com/document/d/1rK7NQK64c53-qbEG-N5xz7uY_QUVI45sUxinbyikCYM/edit).

## JavaScript

### 开源 JS-Fuzzer

JS-Fuzzer是一个基于突变的JavaScript模糊器，最初由Oliver Chang编写。它一直是V8的基石[稳定性](https://bugs.chromium.org/p/chromium/issues/list?q=ochang_js_fuzzer%20label%3AStability-Crash%20label%3AClusterfuzz%20-status%3AWontFix%20-status%3ADuplicate\&can=1)和[安全](https://bugs.chromium.org/p/chromium/issues/list?q=ochang_js_fuzzer%20label%3ASecurity%20label%3AClusterfuzz%20-status%3AWontFix%20-status%3ADuplicate\&can=1)过去和现在[开源](https://chromium-review.googlesource.com/c/v8/v8/+/2320330).

模糊器使用以下命令改变现有的跨引擎测试用例[巴别塔](https://babeljs.io/)可扩展配置的 AST 转换[赋值函数类](https://chromium.googlesource.com/v8/v8/+/320d98709f/tools/clusterfuzz/js_fuzzer/mutators/).我们最近还开始在差分测试模式下运行模糊器的实例，以检测JavaScript。[正确性问题](https://bugs.chromium.org/p/chromium/issues/list?q=blocking%3A1050674%20-status%3ADuplicate\&can=1).欢迎投稿！查看[自述文件](https://chromium.googlesource.com/v8/v8/+/master/tools/clusterfuzz/js_fuzzer/README.md)了解更多。

### 加速`Number.prototype.toString`

在一般情况下，将JavaScript数字转换为字符串可能是一个非常复杂的操作;我们必须考虑浮点精度，科学记数法，NaNs，无穷大，舍入等。在计算之前，我们甚至不知道生成的字符串会有多大。正因为如此，我们的实施`Number.prototype.toString`将拯救到C++运行时函数。

但是，很多时候，您只想打印一个简单的小整数（“Smi”）。这是一个更简单的操作，调用C++运行时函数的开销不再值得。因此，我们与微软的朋友合作，为小整数添加了一个简单的快速路径`Number.prototype.toString`，写在扭矩中，以减少此常见情况下的这些开销。这提高了印刷微基准的数量约75%。

### `Atomics.wake`删除

`Atomics.wake`已重命名为`Atomics.notify`以匹配规范更改[在 v7.3 中](https://v8.dev/blog/v8-release-73#atomics.notify).已弃用的`Atomics.wake`别名现已删除。

### 小的规范性变化

*   匿名类现在有一个`.name`值为空字符串的属性`''`.[规格变更](https://github.com/tc39/ecma262/pull/1490).
*   这`\8`和`\9`转义序列现在在 模板字符串文本中是非法的[草率模式](https://developer.mozilla.org/en-US/docs/Glossary/Sloppy_mode)和 在所有字符串文本中[严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode).[规格变更](https://github.com/tc39/ecma262/pull/2054).
*   内置`Reflect`对象现在有一个`Symbol.toStringTag`值为`'Reflect'`.[规格变更](https://github.com/tc39/ecma262/pull/2057).

## WebAssembly

### 升空时的单色单色

Liftoff是WebAssembly的基准编译器，从V8 v8.5开始，所有平台都发布了。这[SIMD 提案](https://v8.dev/features/simd)使 WebAssembly 能够利用常用的硬件向量指令来加速计算密集型工作负载。它目前在[原产地试用](https://v8.dev/blog/v8-release-84#simd-origin-trial)，这允许开发人员在标准化之前试验某个功能。

到目前为止，SIMD 仅在 V8 的顶级编译器 TurboFan 中实现。这是从 SIMD 指令中获得最佳性能所必需的。与使用 TurboFan 编译的标量等效模块相比，使用 SIMD 指令的 WebAssembly 模块将具有更快的启动速度，并且通常具有更快的运行时性能。例如，给定一个函数，该函数采用浮点数组并将其值钳制为零（为清楚起见，此处用 JavaScript 编写）：

```js
function clampZero(f32array) {
  for (let i = 0; i < f32array.length; ++i) {
    if (f32array[i] < 0) {
      f32array[i] = 0;
    }
  }
}
```

让我们使用Liftoff和TurboFan比较此功能的两种不同实现：

1.  一个标量实现，循环展开 4 次。
2.  SIMD 实现，使用`i32x4.max_s`指令。

使用 Liftoff 标量实现作为基线，我们会看到以下结果：

![A graph showing Liftoff SIMD being ~2.8× faster than Liftoff scalar vs. TurboFan SIMD being ~7.5× faster](/\_img/v8-release-86/simd.svg)

### 更快的 Wasm 到 JS 调用

如果WebAssembly调用导入的JavaScript函数，我们通过所谓的“Wasm-to-JS包装器”（或“import wrapper”）进行调用。此包装器[翻译参数](https://webassembly.github.io/spec/js-api/index.html#tojsvalue)到 JavaScript 理解的对象，当对 JavaScript 的调用返回时，它会转换回返回值[到 WebAssembly](https://webassembly.github.io/spec/js-api/index.html#towebassemblyvalue).

为了保证 JavaScript`arguments`object 准确地反映了从 WebAssembly 传递的参数，如果检测到参数数量不匹配，我们通过所谓的“参数适配器蹦床”进行调用。

但是，在许多情况下，这是不需要的，因为被调用的函数不使用`arguments`对象。在 v8.6 中，我们登陆了一个[补丁](https://crrev.com/c/2317061)由我们的 Microsoft 贡献者提供，在这些情况下避免通过参数适配器进行调用，这使得受影响的调用明显更快。

## V8 接口

### 检测挂起的后台任务`Isolate::HasPendingBackgroundTasks`

新的 API 函数`Isolate::HasPendingBackgroundTasks`允许嵌入者检查是否存在最终将发布新的前台任务的待处理后台工作，例如WebAssembly编译。

这个API应该解决了嵌入器关闭V8的问题，即使仍然有待处理的WebAssembly编译，最终将启动进一步的脚本执行。跟`Isolate::HasPendingBackgroundTasks`嵌入器可以等待新的前台任务，而不是关闭 V8。

请使用`git log branch-heads/8.5..branch-heads/8.6 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.6 -t branch-heads/8.6`以试验 V8 v8.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
