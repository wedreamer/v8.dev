***

标题： 'V8 发布 v8.4'
作者：“卡米洛·布鲁尼，享受一些新鲜的布尔值”
化身：

*   '卡米洛-布鲁尼'
    日期： 2020-06-30
    标签：
*   释放
    描述： “V8 v8.4 具有较弱的引用和改进的 WebAssembly 性能。
    推文：“1277983235641761795”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.4)，直到几周后与Chrome 84 Stable一起发布之前，它一直处于测试阶段。V8 v8.4 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## WebAssembly

### 缩短启动时间

WebAssembly 的基线编译器 （[起飞](https://v8.dev/blog/liftoff)） 现在支持[原子指令](https://github.com/WebAssembly/threads)和[批量内存操作](https://github.com/WebAssembly/bulk-memory-operations).这意味着，即使您使用这些非常新的规格添加，您也可以获得极快的启动时间。

### 更好的调试

为了不断改进 WebAssembly 中的调试体验，我们现在能够在您暂停执行或到达断点时检查任何处于活动状态的 WebAssembly 帧。
这是通过重用实现的[起飞](https://v8.dev/blog/liftoff)用于调试。过去，所有具有断点或单步执行的代码都需要在WebAssembly解释器中执行，这大大减慢了执行速度（通常在100×左右）。使用Liftoff，您只会损失大约三分之一的性能，但您可以随时单步执行所有代码并进行检查。

### 单色原产地试用版

SIMD 提案使 WebAssembly 能够利用常用的硬件矢量指令来加速计算密集型工作负载。V8 具有[支持](https://v8.dev/features/simd)对于[网络组装 SIMD 提案](https://github.com/WebAssembly/simd).要在 Chrome 中启用此功能，请使用标志`chrome://flags/#enable-webassembly-simd`或注册[原产地试验](https://developers.chrome.com/origintrials/#/view_trial/-4708513410415853567).[原产地试验](https://github.com/GoogleChrome/OriginTrials/blob/gh-pages/developer-guide.md)允许开发人员在标准化之前试验功能，并提供有价值的反馈。一旦来源选择加入，试用用户就可以在试用期内选择使用该功能，而无需更新Chrome标志。

## JavaScript

### 弱引用和终结器

：：：备注
**警告！**弱引用和终结器是高级功能！它们依赖于垃圾回收行为。垃圾回收是非确定性的，可能根本不会发生。
:::

JavaScript 是一种垃圾回收语言，这意味着当垃圾回收器运行时，程序无法再访问的对象所占用的内存可能会被自动回收。除 参考文献外`WeakMap`和`WeakSet`，JavaScript 中的所有引用都很强，可以防止被引用的对象被垃圾回收。例如

```js
const globalRef = {
  callback() { console.log('foo'); }
};
// As long as globalRef is reachable through the global scope,
// neither it nor the function in its callback property will be collected.
```

JavaScript程序员现在可以通过`WeakRef`特征。弱引用引用的对象如果没有被强引用，则不会阻止它们被垃圾回收。

```js
const globalWeakRef = new WeakRef({
  callback() { console.log('foo'); }
});

(async function() {
  globalWeakRef.deref().callback();
  // Logs “foo” to console. globalWeakRef is guaranteed to be alive
  // for the first turn of the event loop after it was created.

  await new Promise((resolve, reject) => {
    setTimeout(() => { resolve('foo'); }, 42);
  });
  // Wait for a turn of the event loop.

  globalWeakRef.deref()?.callback();
  // The object inside globalWeakRef may be garbage collected
  // after the first turn since it is not otherwise reachable.
})();
```

的配套功能`WeakRef`s 是`FinalizationRegistry`，这允许程序员注册要在垃圾回收对象后调用的回调。例如，下面的程序可能会记录`42`在收集 IIFE 中无法访问的对象后，添加到控制台。

```js
const registry = new FinalizationRegistry((heldValue) => {
  console.log(heldValue);
});

(function () {
  const garbage = {};
  registry.register(garbage, 42);
  // The second argument is the “held” value which gets passed
  // to the finalizer when the first argument is garbage collected.
})();
```

终结器被安排在事件循环上运行，并且永远不会中断同步 JavaScript 执行。

这些都是先进而强大的功能，如果运气好的话，您的程序将不需要它们。请参阅我们的[解释器](https://v8.dev/features/weak-references)以了解有关它们的更多信息！

### 私有方法和访问器

v7.4 中附带的私有字段通过对私有方法和访问器的支持而得到完善。从语法上讲，私有方法和访问器的名称以`#`，就像私人领域一样。下面简要介绍一下语法。

```js
class Component {
  #privateMethod() {
    console.log("I'm only callable inside Component!");
  }
  get #privateAccessor() { return 42; }
  set #privateAccessor(x) { }
}
```

私有方法和访问器具有与私有字段相同的范围规则和语义。请参阅我们的[解释器](https://v8.dev/features/class-fields)以了解更多信息。

由于[伊加利亚](https://twitter.com/igalia)为实施做出贡献！

## V8 接口

请使用`git log branch-heads/8.3..branch-heads/8.4 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.4 -t branch-heads/8.4`以试验 V8 v8.4 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
