***

标题： 'V8 版本 v9.2'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-07-16
    标签：
*   释放
    描述： 'V8 版本 v9.2 带来了一个`at`用于相对索引和指针压缩改进的方法。
    推特： ''

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.2](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.2)，直到几周后与Chrome 92 Stable合作发布之前，它一直处于测试阶段。V8 v9.2 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `at`方法

新`at`方法现在可用于数组、类型数组和字符串。当传递负值时，它从可索引项的末尾执行相对索引。当传递正值时，它的行为与属性访问相同。例如`[1,2,3].at(-1)`是`3`.查看更多[我们的解释员](https://v8.dev/features/at-method).

## 共享指针压缩笼

V8 支持[指针压缩](https://v8.dev/blog/pointer-compression)在 64 位平台上，包括 x64 和 arm64。这是通过将 64 位指针分成两半来实现的。上面的32位可以被认为是一个基数，而下层的32位可以被认为是该基数的索引。

                |----- 32 bits -----|----- 32 bits -----|
    Pointer:    |________base_______|_______index_______|

目前，隔离器在 4GB 虚拟内存“笼”内执行 GC 堆中的所有分配，这可确保所有指针具有相同的上限 32 位基址。在基址保持恒定的情况下，64 位指针只能使用 32 位索引进行传递，因为可以重建完整指针。

在 v9.2 中，默认设置会更改，以便进程中的所有隔离器共享相同的 4GB 虚拟内存笼。这是为了在JS中对实验性共享内存功能进行原型设计。由于每个工作线程都有自己的隔离器，因此有自己的 4GB 虚拟内存笼，因此指针无法在具有每个隔离笼的隔离之间传递，因为它们不共享相同的基址。此更改还具有在启动工作线程时减少虚拟内存压力的额外好处。

此更改的权衡是，进程中所有线程的总 V8 堆大小上限为最大 4GB。对于每个进程生成许多线程的服务器工作负荷，此限制可能是不希望的，因为这样做会比以前更快地耗尽虚拟内存。嵌入器可能会关闭使用 GN 参数共享指针压缩笼`v8_enable_pointer_compression_shared_cage = false`.

## V8 接口

请使用`git log branch-heads/9.1..branch-heads/9.2 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.2 -t branch-heads/9.2`以试验 V8 v9.2 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。