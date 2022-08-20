***

标题： 'V8 版本 v6.8'
作者： 'V8团队'
日期： 2018-06-21 13：33：37
标签：

*   释放
    描述： “V8 v6.8 的功能降低了内存消耗并提高了多项性能。
    推文：“1009753739060826112”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.8](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.8)，直到几周后与Chrome 68 Stable一起发布之前，它一直处于测试阶段。V8 v6.8 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 记忆

JavaScript 函数不必要地保留了外部函数及其元数据（称为`SharedFunctionInfo`或`SFI`）活着。特别是在依赖于短期 IIFE 的功能繁重的代码中，这可能会导致虚假的内存泄漏。在此更改之前，活动`Context`（即函数激活的堆上表示）保持`SFI`创建上下文的函数处于活动状态：

![](/\_img/v8-release-68/context-jsfunction-before.svg)

通过让`Context`指向`ScopeInfo`对象，其中包含调试所需的精简信息，我们可以打破对`SFI`.

![](/\_img/v8-release-68/context-jsfunction-after.svg)

我们已经观察到，在一组前 10 个页面上，移动设备上的 V8 内存提高了 3%。

同时，我们减少了`SFI`本身，删除不必要的字段或在可能的情况下压缩它们，并将其大小减小了约25%，并在将来的版本中进一步减少。我们观察到`SFI`s 在典型网站上占据了 V8 内存的 2-6%，即使将它们与上下文分离，您也应该看到具有大量函数的代码的内存改进。

## 性能

### 阵列解构改进

优化编译器没有为数组解构生成理想的代码。例如，交换变量`[a, b] = [b, a]`曾经慢到两倍`const tmp = a; a = b; b = tmp`.一旦我们解除了转义分析的阻塞以消除所有临时分配，使用临时数组的数组解构与一系列赋值一样快。

### `Object.assign`改进

迄今`Object.assign`在C++写了一条快速的路径。这意味着必须跨越JavaScript到C++边界`Object.assign`叫。提高内置性能的一个明显方法是在JavaScript端实现快速路径。我们有两个选择：要么将其实现为本机JS内置（在这种情况下会带来一些不必要的开销），要么实现它[使用 CodeStubAssembler 技术](/blog/csa)（这提供了更大的灵活性）。我们采用了后一种解决方案。新实施`Object.assign`提高分数[Speedometer2/React-Redux 提高了约 15%，速度表 2 的总得分提高了 1.5%](https://chromeperf.appspot.com/report?sid=d9ea9a2ae7cd141263fde07ea90da835cf28f5c87f17b53ba801d4ac30979558\&start_rev=550155\&end_rev=552590).

### `TypedArray.prototype.sort`改进

`TypedArray.prototype.sort`有两个路径：快速路径，在用户不提供比较功能时使用，以及用于其他所有内容的慢速路径。到目前为止，慢速路径重用了`Array.prototype.sort`，其作用远远超过排序所需的功能`TypedArray`v8 v6.8 将慢速路径替换为[CodeStubAssembler](/blog/csa).（不是直接的CodeStubAssembler，而是一种基于CodeStubAssembler构建的领域特定语言）。

分拣性能`TypedArray`不带比较函数的 s 保持不变，而使用比较函数进行排序时，加速速度最高可达 2.5×。

![](/\_img/v8-release-68/typedarray-sort.svg)

## WebAssembly

在 V8 v6.8 中，您可以开始使用[基于陷印的边界检查](https://docs.google.com/document/d/17y4kxuHFrVxAiuCP_FFtFA2HP5sNPsCD10KEx17Hz6M/edit)在 Linux x64 平台上。这种内存管理优化大大提高了WebAssembly的执行速度。它已经在Chrome 68中使用，将来将逐步支持更多平台。

## V8 接口

请使用`git log branch-heads/6.7..branch-heads/6.8 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.8 -t branch-heads/6.8`以试验 V8 v6.8 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
