***

标题： 'V8 发布 v9.3'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-08-09
    标签：
*   释放
    描述： 'V8 版本 v9.3 带来了对 Object.hasOwn 和 Error 原因的支持，提高了编译性能，并在 Android 上禁用了不受信任的 codegen 缓解措施。
    推特： ''

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的主Git分支分支分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.3](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.3)，直到几周后与Chrome 93 Stable合作发布之前，它一直处于测试阶段。V8 v9.3 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 火花塞批量编译

我们发布了超快的新中端JIT编译器[火花塞](https://v8.dev/blog/sparkplug)在 v9.1 中。出于安全原因 V8[写保护](https://en.wikipedia.org/wiki/W%5EX)它生成的代码内存，要求它在可写（编译期间）和可执行文件之间切换权限。这是当前使用`mprotect`调用。但是，由于Sparkplug生成代码的速度如此之快，因此调用的成本`mprotect`对于每个单独的编译函数成为编译时间的主要瓶颈。在 V8 v9.3 中，我们为 Sparkplug 引入了批量编译：我们不是单独编译每个函数，而是在一个批处理中编译多个函数。这降低了翻转内存页权限的成本，因为每批只执行一次。

批量编译将整体编译时间（Ignition + Sparkplug）减少了44%，而不会使JavaScript执行倒退。如果我们只看编译Sparkplug代码的成本，影响显然更大，例如，减少82%的`docs_scrolling`Win 10 上的基准测试（见下文）。令人惊讶的是，批处理编译将编译性能提高了比 W^X 更高的成本，因为无论如何，将类似的操作批处理在一起往往对 CPU 更好。在下面的图表中，您可以看到W^X对编译时间（Ignition + Sparkplug）的影响，以及批量编译如何减轻这种开销。

![Benchmarks](../_img/v8-release-93/sparkplug.svg)

### `Object.hasOwn`

`Object.hasOwn`是一个更易于访问的别名`Object.prototype.hasOwnProperty.call`.

例如：

```javascript
Object.hasOwn({ prop: 42 }, 'prop')
// → true
```

稍微多一点（但不是更多！）的详细信息可以在我们的[功能说明](https://v8.dev/features/object-has-own).

### 错误原因

从 v9.3 开始，各种内置`Error`构造函数被扩展为接受带有`cause`属性作为第二个参数。如果传递了这样的选项包，则`cause`属性作为自己的属性安装在`Error`实例。这提供了一种标准化的方法来链接错误。

例如：

```javascript
const parentError = new Error('parent');
const error = new Error('parent', { cause: parentError });
console.log(error.cause === parentError);
// → true
```

像往常一样，请看我们更深入的[功能说明](https://v8.dev/features/error-cause).

## 在 Android 上禁用了不受信任的代码缓解措施

三年前我们推出了一套[代码生成缓解措施](https://v8.dev/blog/spectre)以抵御幽灵的攻击。我们总是意识到，这是一个暂时的权宜之计，只能提供部分保护。[幽灵](https://spectreattack.com/spectre.pdf)攻击。唯一有效的保护是通过以下方式隔离网站[站点隔离](https://blog.chromium.org/2021/03/mitigating-side-channel-attacks.html). 在桌面设备上的 Chrome 上启用网站隔离已有一段时间了，但是由于资源限制，在 Android 上启用完全网站隔离更具挑战性。但是，从Chrome 92开始，[安卓系统上的网站隔离](https://security.googleblog.com/2021/07/protecting-more-with-site-isolation.html)已在包含敏感数据的更多站点上启用。

因此，我们决定在Android上禁用V8的Spectre代码生成缓解措施。这些缓解措施不如站点隔离有效，并且会带来性能成本。禁用它们将使Android与桌面平台相提并论，自V8 v7.0以来它们一直处于关闭状态。通过禁用这些缓解措施，我们看到了Android基准测试性能的一些显着改进。

![Performance improvements](../_img/v8-release-93/code-mitigations.svg)

## V8 接口

请使用`git log branch-heads/9.2..branch-heads/9.3 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.3 -t branch-heads/9.3`以试验 V8 v9.3 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
