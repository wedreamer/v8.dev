***

标题： 'V8 版本 v7.5'
作者：“Dan Elphick，弃用者的祸害”
化身：

*   '丹-埃尔菲克'
    发布日期： 2019-05-16 15：00：00
    标签：
*   释放
    描述： 'V8 v7.5 具有 WebAssembly 编译工件的隐式缓存、批量内存操作、JavaScript 中的数字分隔符等等！
    推文：“1129073370623086593”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.5)，直到几周后与Chrome 75 Stable合作发布之前，它一直处于测试阶段。V8 v7.5 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## WebAssembly

### 隐式缓存

我们计划在 Chrome 75 中推出 WebAssembly 编译工件的隐式缓存。这意味着第二次访问同一页面的用户不需要编译已经看到的WebAssembly模块。相反，它们是从缓存加载的。这类似于[Chromium的JavaScript代码缓存](/blog/code-caching-for-devs).

如果您想在V8嵌入中使用类似的功能，请从Chromium的实现中汲取灵感。

### 批量内存操作

[批量内存建议](https://github.com/webassembly/bulk-memory-operations)向 WebAssembly 添加了新的指令，用于更新大面积的内存或表。

`memory.copy`将数据从一个区域复制到另一个区域，即使这些区域是重叠的（如 C 的`memmove`).`memory.fill`用给定的字节填充区域（如 C 的`memset`).似`memory.copy`,`table.copy`从表的一个区域复制到另一个区域，即使这些区域重叠也是如此。

```wasm
;; Copy 500 bytes from source 1000 to destination 0.
(memory.copy (i32.const 0) (i32.const 1000) (i32.const 500))

;; Fill 1000 bytes starting at 100 with the value `123`.
(memory.fill (i32.const 100) (i32.const 123) (i32.const 1000))

;; Copy 10 table elements from source 5 to destination 15.
(table.copy (i32.const 15) (i32.const 5) (i32.const 10))
```

该提案还提供了一种将常量区域复制到线性内存或表中的方法。为此，我们首先需要定义一个“被动”段。与“活动”段不同，这些段在模块实例化期间不会初始化。相反，可以使用`memory.init`和`table.init`指示。

```wasm
;; Define a passive data segment.
(data $hello passive "Hello WebAssembly")

;; Copy "Hello" into memory at address 10.
(memory.init (i32.const 10) (i32.const 0) (i32.const 5))

;; Copy "WebAssembly" into memory at address 1000.
(memory.init (i32.const 1000) (i32.const 6) (i32.const 11))
```

## JavaScript 中的数字分隔符 { #numeric 分隔符 }

较大的数字文字很难使人眼快速解析，特别是当有很多重复的数字时：

```js
1000000000000
   1019436871.42
```

为了提高可读性，[一个新的 JavaScript 语言特性](/features/numeric-separators)启用下划线作为数字文本中的分隔符。因此，现在可以重写上述内容以对每千个数字进行分组，例如：

```js
1_000_000_000_000
    1_019_436_871.42
```

现在更容易分辨出第一个数字是一万亿，第二个数字是10亿。

有关数字分隔符的更多示例和其他信息，请参阅[我们的解释员](/features/numeric-separators).

## 性能

### 直接从网络流式传输脚本

从Chrome 75开始，V8可以将脚本直接从网络流式传输到流解析器中，而无需等待Chrome主线程。

虽然以前的Chrome版本具有流式解析和编译功能，但由于历史原因，从网络传入的脚本源数据始终必须先进入Chrome主线程，然后再转发到主线程。这意味着，通常，流式解析器会等待已经从网络到达的数据，但尚未转发到流式处理任务，因为它被主线程上发生的其他事情（例如HTML解析，布局或其他JavaScript执行）阻止。

![Stalled background parsing tasks in Chrome 74 and older](../_img/v8-release-75/before.jpg)

在Chrome 75中，我们将网络“数据管道”直接连接到V8，允许我们在流解析期间直接读取网络数据，跳过对主线程的依赖。

![In Chrome 75+, background parsing tasks are no longer blocked by activity on the main thread.](../_img/v8-release-75/after.jpg)

这使我们能够更早地完成流式编译，从而使用流式编译缩短页面的加载时间，并减少并发（但停滞）流式分析任务的数量，从而减少内存消耗。

## V8 接口

请使用`git log branch-heads/7.4..branch-heads/7.5 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.5 -t branch-heads/7.5`以试验 V8 v7.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
