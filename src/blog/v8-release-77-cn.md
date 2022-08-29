***

标题： 'V8 版本 v7.7'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)），发行说明的惰性分配器
化身：

*   'mathias-bynens'
    发布日期： 2019-08-13 16：45：00
    标签：
*   释放
    描述： 'V8 v7.7 具有惰性反馈分配、更快的 WebAssembly 后台编译、堆栈跟踪改进以及新的 International.NumberFormat 功能。
    推文：“1161287541611323397”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.7)，该测试版一直处于测试阶段，直到几周后与Chrome 77 Stable协同发布。V8 v7.7 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 性能（大小和速度）{ #performance }

### 懒惰反馈分配

为了优化JavaScript，V8收集有关传递给各种操作的操作数类型的反馈（例如`+`或`o.foo`).此反馈用于通过针对这些特定类型定制这些操作来优化这些操作。这些信息存储在“反馈向量”中，虽然这些信息对于实现更快的执行时间非常重要，但我们也为分配这些反馈向量所需的内存使用量付出了代价。

为了减少 V8 的内存使用量，我们现在只有在函数执行了一定量的字节码后才懒惰地分配反馈向量。这样可以避免为短期函数分配反馈向量，这些函数不会从收集的反馈中受益。我们的实验室实验表明，懒惰地分配反馈向量可节省约 2–8% 的 V8 堆大小。

![](../_img/v8-release-77/lazy-feedback-allocation.svg)

我们从野外进行的实验表明，对于Chrome用户来说，这可以在桌面设备上将V8的堆大小减少1-2%，在移动平台上减少5-6%。台式机上没有性能下降，在移动平台上，我们实际上在内存有限的低端手机上看到了性能的提高。请留意有关我们最近工作的更详细的博客文章，以节省内存。

### 可扩展的 WebAssembly 后台编译 { #wasm编译 }

在过去的里程碑中，我们研究了WebAssembly后台编译的可扩展性。您的计算机拥有的内核越多，您从这项工作中受益就越多。下图是在 24 核 Xeon 计算机上创建的，正在编译[史诗禅园演示](https://s3.amazonaws.com/mozilla-games/ZenGarden/EpicZenGarden.html).根据所使用的线程数，与 V8 v7.4 相比，编译花费的时间不到一半。

![](../_img/v8-release-77/liftoff-compilation-speedup.svg)

![](../_img/v8-release-77/turbofan-compilation-speedup.svg)

### 堆栈跟踪改进

V8 引发的几乎所有错误在创建时都会捕获堆栈跟踪。这个堆栈跟踪可以从JavaScript通过非标准访问`error.stack`财产。首次检索堆栈跟踪时`error.stack`，V8 将底层结构化堆栈跟踪序列化为字符串。此序列化堆栈跟踪保留下来以加快未来`error.stack`访问。

在过去的几个版本中，我们研究了一些[对堆栈跟踪逻辑的内部重构](https://docs.google.com/document/d/1WIpwLgkIyeHqZBc9D3zDtWr7PL-m_cH6mfjvmoC6kSs/edit)([跟踪错误](https://bugs.chromium.org/p/v8/issues/detail?id=8742)），简化了代码，并将堆栈跟踪序列化性能提高了 30%。

## JavaScript 语言特性

[这`Intl.NumberFormat`应用程序接口](/features/intl-numberformat)用于区域设置感知数字格式，在此版本中获得了新功能！它现在支持紧凑记数法、科学记数法、工程记数法、符号显示和度量单位。

```js
const formatter = new Intl.NumberFormat('en', {
  style: 'unit',
  unit: 'meter-per-second',
});
formatter.format(299792458);
// → '299,792,458 m/s'
```

指[我们的功能解释器](/features/intl-numberformat)了解更多详情。

## V8 接口

请使用`git log branch-heads/7.6..branch-heads/7.7 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.7 -t branch-heads/7.7`以试验 V8 v7.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
