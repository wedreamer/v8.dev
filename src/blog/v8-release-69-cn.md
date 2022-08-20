***

标题： 'V8 版本 v6.9'
作者： 'V8团队'
日期： 2018-08-07 13：33：37
标签：

*   释放
    描述： 'V8 v6.9 通过嵌入式内置功能减少了内存使用量，通过 Liftoff 加快了 WebAssembly 启动速度，提高了 DataView 和 WeakMap 性能，等等！
    推文：“1026825606003150848”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.9)，直到几周后与Chrome 69 Stable一起发布之前，它一直处于测试阶段。V8 v6.9 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 通过嵌入式内置功能节省内存

V8 附带了一个广泛的内置函数库。示例是内置对象上的方法，例如`Array.prototype.sort`和`RegExp.prototype.exec`，还有广泛的内部功能。由于它们的生成需要很长时间，因此内置函数在构建时编译并序列化为[快照](/blog/custom-startup-snapshots)，稍后在运行时反序列化以创建初始 JavaScript 堆状态。

内置函数目前在每个隔离项中的占用 700 KB（隔离项大致对应于 Chrome 中的浏览器标签页）。这是非常浪费的，去年我们开始努力减少这种开销。在 V8 v6.4 中，我们发货[惰性反序列化](/blog/lazy-deserialization)，确保每个隔离软件只为其实际需要的内置功能付费（但每个隔离软件仍然有自己的副本）。

[嵌入式内置](/blog/embedded-builtins)更进一步。嵌入式内置组件由所有隔离器共享，并嵌入到二进制文件本身中，而不是复制到 JavaScript 堆上。这意味着无论运行多少个隔离项，内置组件在内存中只存在一次，这是一个特别有用的属性，现在[站点隔离](https://developers.google.com/web/updates/2018/07/site-isolation)默认情况下已启用。使用嵌入式内置组件，我们已经看到了一个中位数*V8 堆大小减少 9%*在x64上排名前10k的网站上。在这些网站中，50%至少节省1.2 MB，30%节省至少2.1 MB，10%节省3.7 MB或更多。

V8 v6.9 支持 x64 平台上的嵌入式内置功能。其他平台将很快在即将发布的版本中推出。有关更多详细信息，请参阅我们的[专门的博客文章](/blog/embedded-builtins).

## 性能

### Liftoff，WebAssembly新的第一层编译器

WebAssembly获得了一个新的基线编译器，可以更快地启动具有大型WebAssembly模块（如Google Earth和AutoCAD）的复杂网站。根据硬件的不同，我们看到的加速速度超过10×。有关更多详细信息，请参阅[详细的 Liftoff 博客文章](/blog/liftoff).

<figure>
  <img src="/_img/v8-liftoff.svg" width="256" height="256" alt="" loading="lazy">
  <figcaption>Logo for Liftoff, V8’s baseline compiler for WebAssembly</figcaption>
</figure>

### 更快`DataView`操作

[`DataView`](https://tc39.es/ecma262/#sec-dataview-objects)方法已在 V8 Torque 中重新实现，与以前的运行时实现相比，它节省了对 C++ 的昂贵调用。此外，我们现在内联调用`DataView`方法，当在 TurboFan 中编译 JavaScript 代码时，为热代码带来更好的峰值性能。用`DataView`s 现在与使用一样高效`TypedArray`s，最后制作`DataView`在性能关键情况下的可行选择。我们将在即将发布的博客文章中更详细地介绍这一点，关于`DataView`s，敬请期待！

### 更快地处理`WeakMap`垃圾回收期间

V8 v6.9 通过改进来减少 Mark-Compact 垃圾回收暂停时间`WeakMap`加工。并发和增量标记现在能够处理`WeakMap`s，而以前所有这些工作都是在Mark-Compact GC的最后原子暂停中完成的。由于并非所有工作都可以移动到暂停之外，因此GC现在还可以并行执行更多工作，以进一步减少暂停时间。这些优化基本上将 Mark-Compact GC 的平均暂停时间减半[网络工具基准测试](https://github.com/v8/web-tooling-benchmark).

`WeakMap`处理使用定点迭代算法，在某些情况下，该算法可以降级为二次运行时行为。在新版本中，V8 现在能够切换到另一种算法，如果 GC 未在一定次数的迭代内完成，则该算法保证在线性时间内完成。以前，可以构建最坏情况的示例，即使堆相对较小，GC 也需要几秒钟才能完成，而线性算法在几毫秒内完成。

## JavaScript 语言特性

V8 v6.9 支持[`Array.prototype.flat`和`Array.prototype.flatMap`](/features/array-flat-flatmap).

`Array.prototype.flat`以递归方式将给定数组展平到指定的数组`depth`，默认为`1`:

```js
// Flatten one level:
const array = [1, [2, [3]]];
array.flat();
// → [1, 2, [3]]

// Flatten recursively until the array contains no more nested arrays:
array.flat(Infinity);
// → [1, 2, 3]
```

`Array.prototype.flatMap`就像`Array.prototype.map`，只是它将结果平展到新数组中。

```js
[2, 3, 4].flatMap((x) => [x, x * 2]);
// → [2, 4, 3, 6, 4, 8]
```

有关更多详细信息，请参阅[我们`Array.prototype.{flat,flatMap}`解释器](/features/array-flat-flatmap).

## V8 接口

请使用`git log branch-heads/6.8..branch-heads/6.9 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.9 -t branch-heads/6.9`以试验 V8 v6.9 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
