***

标题： 'V8 版本 v9.1'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），测试我的自有品牌'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-05-04
    标签：
*   释放
    描述： 'V8 版本 v9.1 支持自有品牌检查，默认情况下启用顶级 await 和性能改进。
    推文：“1389613320953532417”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.1](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.1)，直到几周后与Chrome 91 Stable一起发布之前，它一直处于测试阶段。V8 v9.1 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `FastTemplateCache`改进

v8 API 公开了一个`Template`嵌入器的接口，从中可以创建新实例。

创建和配置新的对象实例需要几个步骤，这就是为什么克隆现有对象通常更快的原因。V8 使用两级缓存策略（小型快速数组缓存和大型慢速字典缓存）根据模板查找最近创建的对象并直接克隆它们。

以前，模板的缓存索引是在创建模板时分配的，而不是在将模板插入缓存时分配的。这导致快速阵列缓存被保留给通常从未实例化的模板。解决这个问题导致Speedometer2-FlightJS基准测试提高了4.5%。

### 顶级`await`

[顶级`await`](https://v8.dev/features/top-level-await)在 V8 中，从 v9.1 开始，默认情况下处于启用状态，并且在没有`--harmony-top-level-await`.

请注意，对于[眨眼渲染引擎](https://www.chromium.org/blink)，顶级`await`已经[默认启用](https://v8.dev/blog/v8-release-89#top-level-await)在版本 89 中。

嵌入者应注意，通过此启用，`v8::Module::Evaluate`始终返回`v8::Promise`对象而不是完成值。这`Promise`如果模块评估成功，则使用完成值进行解析，如果评估失败，则拒绝并显示错误。如果评估的模块不是异步的（即不包含顶级模块）`await`） 并且没有任何异步依赖项，则返回`Promise`要么被满足，要么被拒绝。否则，返回`Promise`将处于挂起状态。

请参阅[我们的解释员](https://v8.dev/features/top-level-await)了解更多详情。

### 自有品牌支票又名`#foo in obj`

在 v9.1 中，专有品牌检查语法默认处于启用状态，无需`--harmony-private-brand-checks`.此功能扩展了[`in`算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in)以使用私有字段`#`-名称，如以下示例所示。

```javascript
class A {
  static test(obj) {
    console.log(#foo in obj);
  }

  #foo = 0;
}

A.test(new A()); // true
A.test({}); // false
```

要更深入地了解，请务必查看[我们的解释员](https://v8.dev/features/private-brand-checks).

### 短内置调用

在此版本中，我们暂时将未嵌入的内置（撤消[嵌入式内置](https://v8.dev/blog/embedded-builtins)） 在 64 位台式计算机上。在这些计算机上取消嵌入内置组件的性能优势超过了内存成本。这是由于建筑和微观结构细节。

我们将很快发布一篇单独的博客文章，其中包含更多详细信息。

## V8 接口

请使用`git log branch-heads/9.0..branch-heads/9.1 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.1 -t branch-heads/9.1`以试验 V8 v9.1 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
