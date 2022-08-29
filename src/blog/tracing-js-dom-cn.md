***

标题： '从JS到DOM再返回'
作者：“乌兰·德根巴耶夫，阿列克谢·菲利波夫，迈克尔·利普奥茨和汉内斯·佩耶尔 - DOM的团契”
化身：

*   '乌兰-德根巴耶夫'
*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2018-03-01 13：33：37
    标签：
*   内部
*   记忆
    描述： 'Chrome的DevTools现在可以跟踪和快照C++DOM对象，并显示来自JavaScript的所有可访问的DOM对象及其引用。
    推文：“969184997545562112”

***

在Chrome 66中调试内存泄漏变得更加容易。Chrome的DevTools现在可以跟踪和快照C++DOM对象，并显示JavaScript中所有可访问的DOM对象及其引用。此功能是 V8 垃圾回收器新的C++跟踪机制的优点之一。

## 背景

当由于无意中引用其他对象而未释放未使用的对象时，会发生垃圾回收系统中的内存泄漏。网页中的内存泄漏通常涉及 JavaScript 对象和 DOM 元素之间的交互。

以下[玩具示例](https://ulan.github.io/misc/leak.html)显示当程序员忘记注销事件侦听器时发生的内存泄漏。事件侦听器引用的任何对象都不能被垃圾回收。特别是，iframe 窗口与事件侦听器一起泄漏。

```js
// Main window:
const iframe = document.createElement('iframe');
iframe.src = 'iframe.html';
document.body.appendChild(iframe);
iframe.addEventListener('load', function() {
  const localVariable = iframe.contentWindow;
  function leakingListener() {
    // Do something with `localVariable`.
    if (localVariable) {}
  }
  document.body.addEventListener('my-debug-event', leakingListener);
  document.body.removeChild(iframe);
  // BUG: forgot to unregister `leakingListener`.
});
```

泄漏的iframe窗口也保持其所有JavaScript对象处于活动状态。

```js
// iframe.html:
class Leak {};
window.globalVariable = new Leak();
```

了解保留路径的概念以查找内存泄漏的根本原因非常重要。保留路径是一个对象链，用于防止对泄漏对象进行垃圾回收。链从根对象（如主窗口的全局对象）开始。链条在泄漏物体处结束。链中的每个中间对象都有对链中下一个对象的直接引用。例如，保留路径的`Leak`iframe 中的对象如下所示：

![Figure 1: Retaining path of an object leaked via iframe and event listener](../_img/tracing-js-dom/retaining-path.svg)

请注意，保留路径与 JavaScript/DOM 边界（分别以绿色/红色突出显示）相交两次。JavaScript 对象位于 V8 堆中，而 DOM 对象C++对象存在于 Chrome 中。

## DevTools heap snapshot

我们可以通过在 DevTools 中拍摄堆快照来检查任何对象的保留路径。堆快照精确捕获 V8 堆上的所有对象。直到最近，它只有关于C++DOM对象的近似信息。例如，Chrome 65 显示了一个不完整的保留路径`Leak`玩具示例中的对象：

![Figure 2: Retaining path in Chrome 65](../_img/tracing-js-dom/chrome-65.png)

只有第一行是精确的：`Leak`对象确实存储在`global_variable`的 iframe 的窗口对象。后续行近似于实际的保留路径，并使内存泄漏的调试变得困难。

从Chrome 66开始，DevTools通过C++DOM对象进行跟踪，并精确捕获对象和它们之间的引用。这是基于之前为跨组件垃圾回收引入的强大C++对象跟踪机制。因此，[DevTools 中的保留路径](https://www.youtube.com/watch?v=ixadA7DFCx8)现在实际上是正确的：

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/ixadA7DFCx8" width="640" height="360" loading="lazy"></iframe>
  </div>
  <figcaption>Figure 3: Retaining path in Chrome 66</figcaption>
</figure>

## 幕后：跨组件跟踪

DOM对象由Blink管理 ，Blink是Chrome的渲染引擎，负责将DOM转换为屏幕上的实际文本和图像。Blink及其对DOM的表示是用C++编写的，这意味着DOM不能直接暴露给JavaScript。相反，DOM 中的对象分为两半：可用于 JavaScript 的 V8 包装器对象和表示 DOM 中节点的C++对象。这些对象彼此直接引用。跨多个组件（如 Blink 和 V8）确定对象的活动性和所有权是很困难的，因为所有相关方都需要就哪些对象仍然处于活动状态以及哪些对象可以回收达成一致。

在Chrome 56及更早的版本（即直到2017年3月之前），Chrome使用了一种称为*对象分组*以确定活动性。根据文档中的包含情况为对象分配组。只要单个对象通过其他保留路径保持活动状态，具有所有包含对象的组就会保持活动状态。这在 DOM 节点的上下文中是有意义的，这些节点总是引用其包含的文档，从而形成所谓的 DOM 树。但是，此抽象删除了所有实际的保留路径，这使得它难以用于调试，如图 2 所示。对于不适合这种情况的对象，例如用作事件侦听器的 JavaScript 闭包，这种方法也变得繁琐，并导致各种 Bug，其中 JavaScript 包装器对象会过早地被收集，这导致它们被空的 JS 包装器替换，这些包装器将丢失其所有属性。

从Chrome 57开始，这种方法被跨组件跟踪所取代，这是一种通过将JavaScript跟踪到DOM的C++实现并返回来确定活动性的机制。我们在C++端实施了增量跟踪，并设置了写入障碍，以避免我们一直在谈论的任何停止世界跟踪卡顿[以前的博客文章](/blog/orinoco-parallel-scavenger).跨组件跟踪不仅提供更好的延迟，而且可以更好地跨组件边界近似对象的活动性，并修复了几个[场景](https://bugs.chromium.org/p/chromium/issues/detail?id=501866)曾经导致泄漏。最重要的是，它允许 DevTools 提供实际表示 DOM 的快照，如图 3 所示。

试试吧！我们很高兴听到您的反馈。
