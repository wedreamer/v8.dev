***

标题： 'V8 发布 v8.3'
作者： '[维克多·戈麦斯](https://twitter.com/VictorBFG)，在家安全工作'
化身：

*   “胜利者戈麦斯”
    日期： 2020-05-04
    标签：
*   释放
    描述： “V8 v8.3 具有更快的 ArrayBuffers、更大的 Wasm 内存和已弃用的 API。
    推文：“1257333120115847171”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.3](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.3)，直到几周后与Chrome 83 Stable合作发布之前，它一直处于测试阶段。V8 v8.3 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 性能

### 更快`ArrayBuffer`垃圾回收器中的跟踪

后备商店`ArrayBuffer`s 在 V8 的堆外分配，使用`ArrayBuffer::Allocator`由嵌入器提供。这些后备商店需要在他们的`ArrayBuffer`对象由垃圾回收器回收。V8 v8.3 有了新的跟踪机制`ArrayBuffer`及其后备存储，允许垃圾回收器迭代并释放后备存储并同时释放到应用程序。更多详细信息，请参见[本设计文档](https://docs.google.com/document/d/1-ZrLdlFX1nXT3z-FAgLbKal1gI8Auiaya_My-a0UJ28/edit#heading=h.gfz6mi5p212e).这减少了 GC 的总暂停时间`ArrayBuffer`繁重的工作负载增加了 50%。

### 更大的瓦斯姆回忆

根据更新[网页组件规范](https://webassembly.github.io/spec/js-api/index.html#limits)，V8 v8.3 现在允许模块请求大小高达 4GB 的内存，从而允许将更多内存密集型用例引入由 V8 提供支持的平台。请记住，在用户的系统上可能并不总是可用的这么多内存;我们建议创建较小大小的内存，根据需要增加它们，并优雅地处理增长失败。

## 修复

### 存储到原型链上具有类型化数组的对象

根据JavaScript规范，当将值存储到指定的键时，我们需要查找原型链以查看原型上是否已经存在该键。通常，这些键在原型链上不存在，因此V8会安装快速查找处理程序，以避免这些原型链在安全的情况下行走。

但是，我们最近发现了一个特殊情况，其中 V8 错误地安装了此快速查找处理程序，从而导致不正确的行为。什么时候`TypedArray`s 在原型链上，所有存储到密钥，这些密钥是 OOB 的`TypedArray`应该被忽略。例如，在下面的例子中`v[2]`不应将属性添加到`v`并且后续读取应返回未定义。

```js
v = {};
v.__proto__ = new Int32Array(1);
v[2] = 123;
return v[2]; // Should return undefined
```

V8 的快速查找处理程序无法处理这种情况，我们会返回`123`在上面的例子中。V8 v8.3 通过在以下情况下不使用快速查找处理程序来解决此问题`TypedArray`s 在原型链上。鉴于这不是一个常见的情况，我们在基准测试中没有看到任何性能回归。

## V8 接口

### 实验性 WeakRefs 和 FinalizationRegistry API 已弃用

以下与 WeakRefs 相关的实验性 API 已弃用：

*   `v8::FinalizationGroup`
*   `v8::Isolate::SetHostCleanupFinalizationGroupCallback`

`FinalizationRegistry`（重命名自`FinalizationGroup`） 是[JavaScript 弱引用提案](https://v8.dev/features/weak-references)并为JavaScript程序员提供了一种注册终结器的方法。这些 API 供嵌入器计划和运行`FinalizationRegistry`调用已注册终结器的清理任务;它们已被弃用，因为它们不再需要。`FinalizationRegistry`清理任务现在由 V8 使用嵌入器提供的前台任务运行程序自动调度`v8::Platform`并且不需要任何其他嵌入器代码。

### 其他 API 更改

请使用`git log branch-heads/8.1..branch-heads/8.3 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.3 -t branch-heads/8.3`以试验 V8 v8.3 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
