***

标题： 'V8 版本 v7.2'
作者：“安德烈亚斯·哈斯，陷阱处理者”
化身：

*   安德烈亚斯-哈斯
    日期： 2018-12-18 11：48：21
    标签：
*   释放
    描述： 'V8 v7.2 具有高速 JavaScript 解析、更快的异步等待、减少 ia32 上的内存消耗、公共类字段等等！
    推文：“1074978755934863361”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.2](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.2)，直到几周后与Chrome 72 Stable一起发布之前，它一直处于测试阶段。V8 v7.2 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 记忆

[嵌入式内置](/blog/embedded-builtins)现在在 ia32 体系结构上默认受支持并启用。

## 性能

### JavaScript 解析

平均而言，网页在启动时花费9.5%的V8时间来解析JavaScript。因此，我们专注于在 v7.2 中发布 V8 最快的 JavaScript 解析器。我们全面提高了解析速度。自 v7.0 起，桌面上的解析速度提高了约 30%。下图记录了过去几个月来我们实际Facebook加载基准测试的令人印象深刻的改进。

![V8 parse time on facebook.com (lower is better)](../_img/v8-release-72/facebook-parse-time.png)

我们在不同的场合专注于解析器。下图显示了几个热门网站相对于最新 v7.2 版本的改进。

![V8 parse times relative to V8 v7.2 (lower is better)](../_img/v8-release-72/relative-parse-times.svg)

总而言之，最近的改进已将平均解析百分比从 9.5% 降低到 7.5%，从而缩短了加载速度，提高了页面响应速度。

### `async`/`await`

V8 v7.2 附带[更快`async`/`await`实现](/blog/fast-async#await-under-the-hood)，默认情况下处于启用状态。我们已使[规范建议](https://github.com/tc39/ecma262/pull/1250)并且目前正在收集 Web 兼容性数据，以便将更改正式合并到 ECMAScript 规范中。

### 展开元素

V8 v7.2 大大提高了展开元素在数组文本前面出现时的性能，例如`[...x]`或`[...x, 1, 2]`.此改进适用于分散数组、基元字符串、集合、映射键、映射值，以及 （ 通过扩展 ）`Array.from(x)`.有关更多详细信息，请参阅[我们关于加速传播元素的深入文章](/blog/spread-elements).

### WebAssembly

我们分析了许多 WebAssembly 基准测试，并用它们来指导在顶级执行层中改进代码生成。特别是，V8 v7.2 在优化编译器的调度程序中支持节点拆分，在后端实现循环循环。我们还改进了包装器缓存，并引入了自定义包装器，以减少调用导入的 JavaScript 数学函数的开销。此外，我们还设计了对寄存器分配器的更改，以提高许多代码模式的性能，这些代码模式将登陆更高版本。

### 陷阱处理程序

陷阱处理程序正在提高 WebAssembly 代码的一般吞吐量。它们在V8 v7.2的Windows，macOS和Linux上实现并可用。在Chromium中，它们在Linux上是启用的。当稳定性得到确认时，Windows和macOS将效仿。我们目前也正在努力使它们在Android上可用。

## 异步堆栈跟踪

如[前面提到过](/blog/fast-async#improved-developer-experience)，我们添加了一个名为[零成本异步堆栈跟踪](https://bit.ly/v8-zero-cost-async-stack-traces)，这丰富了`error.stack`具有异步调用帧的属性。它目前在`--async-stack-traces`命令行标志。

## JavaScript 语言特性

### 公共类字段

V8 v7.2 增加了对[公共类字段](/features/class-fields).而不是：

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}

class Cat extends Animal {
  constructor(name) {
    super(name);
    this.likesBaths = false;
  }
  meow() {
    console.log('Meow!');
  }
}
```

...你现在可以写：

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}

class Cat extends Animal {
  likesBaths = false;
  meow() {
    console.log('Meow!');
  }
}
```

支持[私有类字段](/features/class-fields#private-class-fields)计划在将来的 V8 版本中发布。

### `Intl.ListFormat`

V8 v7.2 增加了对[这`Intl.ListFormat`建议](/features/intl-listformat)，启用列表的本地化格式。

```js
const lf = new Intl.ListFormat('en');
lf.format(['Frank']);
// → 'Frank'
lf.format(['Frank', 'Christine']);
// → 'Frank and Christine'
lf.format(['Frank', 'Christine', 'Flora']);
// → 'Frank, Christine, and Flora'
lf.format(['Frank', 'Christine', 'Flora', 'Harrison']);
// → 'Frank, Christine, Flora, and Harrison'
```

有关详细信息和使用示例，请查看[我们`Intl.ListFormat`解释器](/features/intl-listformat).

### 格式良好`JSON.stringify`

`JSON.stringify`现在输出单独代理项的转义序列，使其输出有效的 Unicode（并且可用 UTF-8 表示）：

```js
// Old behavior:
JSON.stringify('\uD800');
// → '"�"'

// New behavior:
JSON.stringify('\uD800');
// → '"\\ud800"'
```

有关详细信息，请参阅[我们精心打造`JSON.stringify`解释器](/features/well-formed-json-stringify).

### 模块命名空间导出

在[JavaScript 模块](/features/modules)，则已经可以使用以下语法：

```js
import * as utils from './utils.mjs';
```

但是，没有对称`export`语法存在...[直到现在](/features/module-namespace-exports):

```js
export * as utils from './utils.mjs';
```

这等效于以下内容：

```js
import * as utils from './utils.mjs';
export { utils };
```

## V8 接口

请使用`git log branch-heads/7.1..branch-heads/7.2 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.2 -t branch-heads/7.2`以试验 V8 v7.2 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
