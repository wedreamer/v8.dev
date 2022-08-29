***

标题： 'V8 版本 v7.6'
作者： '亚当·克莱因'
化身：

*   “亚当-克莱因”
    发布日期： 2019-06-19 16：45：00
    标签：
*   释放
    描述： 'V8 v7.6 功能 Promise.allSettled， 更快的 JSON.parse， 本地化的 BigInts， 更快的冻结/密封数组， 以及更多！
    推文：“1141356209179516930”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.6)，直到几周后与Chrome 76 Stable合作发布之前，它一直处于测试阶段。V8 v7.6 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 性能（大小和速度）{ #performance }

### `JSON.parse`改进

在现代JavaScript应用程序中，JSON通常用作传达结构化数据的格式。通过加速JSON解析，我们可以减少这种通信的延迟。在 V8 v7.6 中，我们对 JSON 解析器进行了全面修改，使其在扫描和解析 JSON 时速度更快。这样可以更快地解析热门网页提供的数据× 2.7.7。

![Chart showing improved performance of JSON.parse on a variety of websites](../_img/v8-release-76/json-parsing.svg)

在 V8 v7.5 之前，JSON 解析器是一个递归解析器，它将使用相对于传入 JSON 数据的嵌套深度的本机堆栈空间。这意味着我们可能会用完非常深嵌套的JSON数据的堆栈。V8 v7.6 切换到一个迭代解析器，该解析器管理自己的堆栈，该堆栈仅受可用内存的限制。

新的 JSON 解析器也提高了内存效率。通过在创建最终对象之前缓冲属性，我们现在可以决定如何以最佳方式分配结果。对于具有命名属性的对象，我们为传入 JSON 数据中的命名属性分配具有确切空间量的对象（最多 128 个命名属性）。如果 JSON 对象包含索引属性名称，我们分配一个使用最小空间量的元素后备存储;平面数组或字典。JSON 数组现在被解析为完全适合输入数据中元素数的数组。

### 冻结/密封阵列改进

对冻结或密封数组（和类似数组的对象）的调用性能得到了许多改进。V8 v7.6 提升了以下 JavaScript 编码模式，其中`frozen`是冻结或密封的数组或类似数组的对象：

*   `frozen.indexOf(v)`
*   `frozen.includes(v)`
*   传播呼叫，例如`fn(...frozen)`
*   使用嵌套数组展开扩展调用，例如`fn(...[...frozen])`
*   应用具有数组分布的调用，例如`fn.apply(this, [...frozen])`

下图显示了改进。

![Chart showing performance boost on a variety of array operations](../_img/v8-release-76/frozen-sealed-elements.svg)

[请参阅“V8 中的快速冷冻和密封元件”设计文档](https://bit.ly/fast-frozen-sealed-elements-in-v8)了解更多详情。

### 统一码字符串处理

优化时[将字符串转换为 Unicode](https://chromium.googlesource.com/v8/v8/+/734c1456d942a03d79aab4b3b0e57afbc803ceea)导致呼叫的显着加速，例如`String#localeCompare`,`String#normalize`，以及一些`Intl`蜜蜂属。例如，此更改导致原始吞吐量约为 2×`String#localeCompare`对于单字节字符串。

## JavaScript 语言特性

### `Promise.allSettled`

[`Promise.allSettled(promises)`](/features/promise-combinators#promise.allsettled)当所有输入承诺为*安定*，这意味着它们要么*实现*或*拒绝*.这在你不关心承诺的状态的情况下很有用，你只想知道工作何时完成，无论它是否成功。[我们的解释器关于承诺组合器](/features/promise-combinators)包含更多详细信息，并包含一个示例。

### 改进`BigInt`支持 { #localized-bigint }

[`BigInt`](/features/bigint)现在在语言中具有更好的API支持。您现在可以格式化`BigInt`通过使用`toLocaleString`方法。这就像常规数字一样：

```js
12345678901234567890n.toLocaleString('en'); // 🐌
// → '12,345,678,901,234,567,890'
12345678901234567890n.toLocaleString('de'); // 🐌
// → '12.345.678.901.234.567.890'
```

如果您计划设置多个数字的格式，或者`BigInt`s 使用相同的区域设置，使用`Intl.NumberFormat`API，现在支持`BigInt`s 在其`format`和`formatToParts`方法。这样，您可以创建单个可重用的格式化程序实例。

```js
const nf = new Intl.NumberFormat('fr');
nf.format(12345678901234567890n); // 🚀
// → '12 345 678 901 234 567 890'
nf.formatToParts(123456n); // 🚀
// → [
// →   { type: 'integer', value: '123' },
// →   { type: 'group', value: ' ' },
// →   { type: 'integer', value: '456' }
// → ]
```

### `Intl.DateTimeFormat`改进 { #intl-日期时间格式 }

应用通常显示日期间隔或日期范围，以显示事件的跨度，例如酒店预订、服务的计费周期或音乐节。这`Intl.DateTimeFormat`API 现在支持`formatRange`和`formatRangeToParts`方法，以特定于区域设置的方式方便地设置日期范围的格式。

```js
const start = new Date('2019-05-07T09:20:00');
// → 'May 7, 2019'
const end = new Date('2019-05-09T16:00:00');
// → 'May 9, 2019'
const fmt = new Intl.DateTimeFormat('en', {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
});
const output = fmt.formatRange(start, end);
// → 'May 7 – 9, 2019'
const parts = fmt.formatRangeToParts(start, end);
// → [
// →   { 'type': 'month',   'value': 'May',  'source': 'shared' },
// →   { 'type': 'literal', 'value': ' ',    'source': 'shared' },
// →   { 'type': 'day',     'value': '7',    'source': 'startRange' },
// →   { 'type': 'literal', 'value': ' – ',  'source': 'shared' },
// →   { 'type': 'day',     'value': '9',    'source': 'endRange' },
// →   { 'type': 'literal', 'value': ', ',   'source': 'shared' },
// →   { 'type': 'year',    'value': '2019', 'source': 'shared' },
// → ]
```

此外，`format`,`formatToParts`和`formatRangeToParts`方法现在支持新的`timeStyle`和`dateStyle`选项：

```js
const dtf = new Intl.DateTimeFormat('de', {
  timeStyle: 'medium',
  dateStyle: 'short'
});
dtf.format(Date.now());
// → '19.06.19, 13:33:37'
```

## 原生堆栈行走

虽然 V8 可以遍历自己的调用堆栈（例如，在 DevTools 中调试或分析时），但 Windows 操作系统无法在 x64 架构上运行时遍历包含 TurboFan 生成的代码的调用堆栈。这可能会导致*破碎的堆栈*使用本机调试器或 ETW 采样来分析使用 V8 的进程时。最近的更改使 V8 能够[注册必要的元数据](https://chromium.googlesource.com/v8/v8/+/3cda21de77d098a612eadf44d504b188a599c5f0)对于 Windows，以便能够在 x64 上遍历这些堆栈，而在 v7.6 中，默认情况下启用此功能。

## V8 接口

请使用`git log branch-heads/7.5..branch-heads/7.6 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.6 -t branch-heads/7.6`以试验 V8 v7.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
