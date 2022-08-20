***

标题：“弱引用和终结器”
作者： 'Sathya Gunasekaran （[@\_gsathya](https://twitter.com/\_gsathya)）， 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)）， 郭淑宇 （[@\_shu](https://twitter.com/\_shu)），和 Leszek Swirski （[@leszekswirski](https://twitter.com/leszekswirski))'
化身：

*   'sathya-gunasekaran'
*   'mathias-bynens'
*   “舒玉国”
*   'leszek-swirski'
    日期： 2019-07-09
    更新时间：2020 年 6 月 19 日
    标签：
    *   ECMAScript
    *   ES2021
    *   io19
    *   节点.js 14
        描述： '弱引用和终结器即将来到 JavaScript！本文将介绍新功能。
        推文：“1148603966848151553”

***

通常，对对象的引用是*强势持有*在JavaScript中，这意味着只要你有对对象的引用，它就不会被垃圾回收。

```js
const ref = { x: 42, y: 51 };
// As long as you have access to `ref` (or any other reference to the
// same object), the object won’t be garbage-collected.
```

现在`WeakMap`s 和`WeakSet`s 是在 JavaScript 中弱引用对象的唯一方法：将对象添加为`WeakMap`或`WeakSet`不会阻止它被垃圾回收。

```js
const wm = new WeakMap();
{
  const ref = {};
  const metaData = 'foo';
  wm.set(ref, metaData);
  wm.get(ref);
  // → metaData
}
// We no longer have a reference to `ref` in this block scope, so it
// can be garbage-collected now, even though it’s a key in `wm` to
// which we still have access.

const ws = new WeakSet();
{
  const ref = {};
  ws.add(ref);
  ws.has(ref);
  // → true
}
// We no longer have a reference to `ref` in this block scope, so it
// can be garbage-collected now, even though it’s a key in `ws` to
// which we still have access.
```

：：：备注
**注意：**你可以想到`WeakMap.prototype.set(ref, metaData)`作为添加具有值的属性`metaData`到对象`ref`：只要您有对对象的引用，就可以获取元数据。一旦您不再具有对该对象的引用，即使它仍然具有对`WeakMap`它被添加到其中。同样，你可以想到一个`WeakSet`作为特殊情况`WeakMap`其中，所有值都是布尔值。

一个 JavaScript`WeakMap`不是真的*弱*：它实际上指的是*强烈*只要密钥还活着，就可以达到其内容。这`WeakMap`只有在密钥被垃圾回收后，才弱地引用其内容。这种关系的更准确的名称是[*短暂*](https://en.wikipedia.org/wiki/Ephemeron).
:::

`WeakRef`是一个更高级的 API，它提供*实际*弱引用，启用一个窗口，了解对象的生存期。让我们一起看一个示例。

对于此示例，假设我们正在开发一个聊天 Web 应用程序，该应用程序使用 Web 套接字与服务器进行通信。想象一下`MovingAvg`类，出于性能诊断目的，从 Web 套接字中保留一组事件，以便计算延迟的简单移动平均值。

```js
class MovingAvg {
  constructor(socket) {
    this.events = [];
    this.socket = socket;
    this.listener = (ev) => { this.events.push(ev); };
    socket.addEventListener('message', this.listener);
  }

  compute(n) {
    // Compute the simple moving average for the last n events.
    // …
  }
}
```

它由`MovingAvgComponent`类，允许您控制何时开始和停止监视延迟的简单移动平均值。

```js
class MovingAvgComponent {
  constructor(socket) {
    this.socket = socket;
  }

  start() {
    this.movingAvg = new MovingAvg(this.socket);
  }

  stop() {
    // Allow the garbage collector to reclaim memory.
    this.movingAvg = null;
  }

  render() {
    // Do rendering.
    // …
  }
}
```

我们知道将所有服务器消息保存在一个实例中`MovingAvg`使用大量内存，因此我们注意清空`this.movingAvg`当停止监视以让垃圾回收器回收内存时。

但是，在DevTools中检查内存面板后，我们发现根本没有回收内存！经验丰富的Web开发人员可能已经发现了这个错误：事件侦听器是强引用，必须显式删除。

让我们通过可访问性图来明确这一点。呼叫后`start()`，我们的对象图如下所示，其中实心箭头表示强引用。一切都可以通过实心箭头到达`MovingAvgComponent`实例不可垃圾回收。

![](/\_img/weakrefs/after-start.svg)

呼叫后`stop()`，我们已从`MovingAvgComponent`实例到`MovingAvg`实例，但不是通过套接字的侦听器。

![](/\_img/weakrefs/after-stop.svg)

因此，听众在`MovingAvg`实例，通过引用`this`，只要不删除事件侦听器，整个实例就会保持活动状态。

到目前为止，解决方案是通过`dispose`方法。

```js
class MovingAvg {
  constructor(socket) {
    this.events = [];
    this.socket = socket;
    this.listener = (ev) => { this.events.push(ev); };
    socket.addEventListener('message', this.listener);
  }

  dispose() {
    this.socket.removeEventListener('message', this.listener);
  }

  // …
}
```

这种方法的缺点是它是手动内存管理。`MovingAvgComponent`，以及`MovingAvg`上课，一定要记得打电话`dispose`或遭受内存泄漏。更糟糕的是，手动内存管理是级联的：用户`MovingAvgComponent`一定要记得打电话`stop`或者遭受内存泄漏，等等。应用程序行为不依赖于此诊断类的事件侦听器，并且侦听器在内存使用方面成本高昂，但在计算方面不昂贵。我们真正想要的是让听众的一生在逻辑上与`MovingAvg`实例，以便`MovingAvg`可以像任何其他JavaScript对象一样使用，其内存由垃圾回收器自动回收。

`WeakRef`s 通过创建一个*弱引用*到实际的事件侦听器，然后将其包装`WeakRef`在外部事件侦听器中。这样，垃圾回收器就可以清理实际的事件侦听器及其处于活动状态的内存，如`MovingAvg`实例及其`events`数组。

```js
function addWeakListener(socket, listener) {
  const weakRef = new WeakRef(listener);
  const wrapper = (ev) => { weakRef.deref()?.(ev); };
  socket.addEventListener('message', wrapper);
}

class MovingAvg {
  constructor(socket) {
    this.events = [];
    this.listener = (ev) => { this.events.push(ev); };
    addWeakListener(socket, this.listener);
  }
}
```

：：：备注
**注意：** `WeakRef`s 到函数必须谨慎对待。JavaScript 函数是[闭 包](https://en.wikipedia.org/wiki/Closure_\(computer_programming\))并强烈引用包含函数内部引用的自由变量值的外部环境。这些外部环境可能包含以下变量：*其他*闭包引用也是如此。也就是说，在处理闭包时，它们的记忆通常以微妙的方式被其他闭包强烈引用。这就是原因`addWeakListener`是一个单独的函数，并且`wrapper`不是本地的`MovingAvg`构造 函数。在 V8 中，如果`wrapper`是本地的`MovingAvg`构造函数，并与包装在`WeakRef`这`MovingAvg`实例及其所有属性都可以通过包装器侦听器的共享环境进行访问，从而导致实例无法收集。在编写代码时请记住这一点。
:::

我们首先创建事件侦听器并将其分配给`this.listener`，以便它被`MovingAvg`实例。换句话说，只要`MovingAvg`实例处于活动状态，事件侦听器也处于活动状态。

然后，在`addWeakListener`，我们创建一个`WeakRef`谁的*目标*是实际的事件侦听器。里面`wrapper`我们`deref`它。因为`WeakRef`如果目标没有其他强引用，则不会阻止对其目标进行垃圾回收，我们必须手动取消引用它们才能获得目标。如果在此期间对目标进行了垃圾回收，`deref`返回`undefined`.否则，将返回原始目标，即`listener`然后我们使用调用函数[可选链接](/features/optional-chaining).

由于事件侦听器包装在`WeakRef`这*只*强力引用它是`listener`属性`MovingAvg`实例。也就是说，我们已经成功地将事件侦听器的生存期与`MovingAvg`实例。

返回到可访问性图，调用后，我们的对象图如下所示`start()`与`WeakRef`实现，其中虚线箭头表示弱引用。

![](/\_img/weakrefs/weak-after-start.svg)

呼叫后`stop()`，我们删除了对侦听器的唯一强引用：

![](/\_img/weakrefs/weak-after-stop.svg)

最终，在发生垃圾回收后，`MovingAvg`实例和侦听器将被收集：

![](/\_img/weakrefs/weak-after-gc.svg)

但这里仍然存在一个问题：我们增加了一定程度的间接寻址`listener`通过包装它`WeakRef`，但包装器`addWeakListener`由于同样的原因，仍然泄漏`listener`最初泄漏。当然，这是一个较小的泄漏，因为只有包装器而不是整个包装器泄漏`MovingAvg`实例，但它仍然是泄漏。对此的解决方案是配套功能`WeakRef`,`FinalizationRegistry`.随着新的`FinalizationRegistry`API，我们可以注册一个回调，以便在垃圾回收器删除寄存器对象时运行。此类回调称为*终结器*.

：：：备注
**注意：**在对事件侦听器进行垃圾回收后，终结回调不会立即运行，因此不要将其用于重要的逻辑或指标。垃圾回收和终结回调的时间未指定。事实上，一个从不进行垃圾回收的引擎是完全兼容的。但是，可以安全地假设发动机*将*垃圾回收和终结回调将在稍后某个时间调用，除非环境被丢弃（例如选项卡关闭或工作线程终止）。在编写代码时，请记住这种不确定性。
:::

我们可以注册一个回调`FinalizationRegistry`以删除`wrapper`当内部事件侦听器被垃圾回收时，从套接字。我们的最终实现如下所示：

```js
const gListenersRegistry = new FinalizationRegistry(({ socket, wrapper }) => {
  socket.removeEventListener('message', wrapper); // 6
});

function addWeakListener(socket, listener) {
  const weakRef = new WeakRef(listener); // 2
  const wrapper = (ev) => { weakRef.deref()?.(ev); }; // 3
  gListenersRegistry.register(listener, { socket, wrapper }); // 4
  socket.addEventListener('message', wrapper); // 5
}

class MovingAvg {
  constructor(socket) {
    this.events = [];
    this.listener = (ev) => { this.events.push(ev); }; // 1
    addWeakListener(socket, this.listener);
  }
}
```

：：：备注
**注意：** `gListenersRegistry`是一个全局变量，用于确保执行终结器。一个`FinalizationRegistry`不被在其上注册的对象保持活动状态。如果注册表本身是垃圾回收的，则其终结器可能无法运行。
:::

我们创建一个事件侦听器并将其分配给`this.listener`因此，它被`MovingAvg`实例 （1）。然后，我们将执行工作的事件侦听器包装在`WeakRef`使其可垃圾收集，并且不泄漏其对`MovingAvg`实例通过`this`（2）.我们制作了一个包装器`deref`这`WeakRef`检查它是否还活着，如果是这样，则调用它（3）。我们在`FinalizationRegistry`，传递一个*持有价值* `{ socket, wrapper }`到注册 （4）。然后，我们将返回的包装器添加为事件侦听器`socket`（5）.以后的某个时候`MovingAvg`实例和内部侦听器是垃圾回收的，终结器可以运行，并保持值传递给它。在终结器内部，我们还删除了包装器，使所有内存都与使用`MovingAvg`实例可垃圾回收 （6）。

有了这一切，我们原来实施`MovingAvgComponent`既不泄漏内存，也不需要任何手动处理。

## 不要过度使用 { #progressive增强 }

在听说了这些新功能之后，可能会很诱人`WeakRef`所有的事情™。但是，这可能不是一个好主意。有些事情是明确的*不*良好的用例`WeakRef`s 和终结器。

通常，避免编写依赖于垃圾回收器清理`WeakRef`或者在任何可预测的时间调用终结器 —[它不能完成](https://github.com/tc39/proposal-weakrefs#a-note-of-caution)!此外，一个对象是否是垃圾回收的可能取决于实现细节，例如闭包的表示，这些细节既微妙又可能不同，并且可能在JavaScript引擎之间甚至同一引擎的不同版本之间有所不同。具体而言，终结器回调：

*   垃圾回收后可能不会立即发生。
*   可能不会以与实际垃圾回收相同的顺序发生。
*   可能根本不会发生，例如，如果浏览器窗口已关闭。

因此，不要将重要的逻辑放在终结器的代码路径中。它们对于执行清理以响应垃圾回收非常有用，但您无法可靠地使用它们来记录有关内存使用情况的有意义的指标。有关该用例，请参阅[`performance.measureUserAgentSpecificMemory`](https://web.dev/monitor-total-page-memory-usage/).

`WeakRef`s 和终结器可以帮助您节省内存，并在有节制地用作渐进式增强的手段时效果最佳。由于它们是高级用户功能，因此我们预计大多数使用都发生在框架或库中。

## `WeakRef`支持 { #support }

<feature-support chrome="74 https://v8.dev/blog/v8-release-84#weak-references-and-finalizers"
              firefox="79 https://bugzilla.mozilla.org/show_bug.cgi?id=1561074"
              safari="no"
              nodejs="14.6.0"
              babel="no"></feature-support>
