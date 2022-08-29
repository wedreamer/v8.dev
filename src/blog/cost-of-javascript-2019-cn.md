***

标题：“2019 年 JavaScript 的成本”
作者： '阿迪·奥斯曼尼 （[@addyosmani](https://twitter.com/addyosmani)），JavaScript Janitor和Mathias Bynens（[@mathias](https://twitter.com/mathias)），主线程解放者
化身：

*   'addy-osmani'
*   'mathias-bynens'
    日期： 2019-06-25
    标签：
*   内部
*   解析
    描述：“处理 JavaScript 的主要成本是下载和 CPU 执行时间。
    推文：“1143531042361487360”

***

：：：备注
**注意：**如果您更喜欢观看演示文稿而不是阅读文章，那么请欣赏下面的视频！如果没有，请跳过视频并继续阅读。
:::

<figure>
  <div class="video video-16:9">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/X9eRLElSW1c" allow="picture-in-picture" allowfullscreen loading="lazy"></iframe>
  </div>
  <figcaption><a href="https://www.youtube.com/watch?v=X9eRLElSW1c">“The cost of JavaScript”</a> as presented by Addy Osmani at #PerfMatters Conference 2019.</figcaption>
</figure>

一个大的变化[JavaScript 的成本](https://medium.com/@addyosmani/the-cost-of-javascript-in-2018-7d8950fbb5d4)在过去几年中，浏览器解析和编译脚本的速度有所提高。**在2019年，处理脚本的主要成本现在是下载和CPU执行时间。**

如果浏览器的主线程忙于执行 JavaScript，则用户交互可能会延迟，因此优化脚本执行时间和网络的瓶颈可能会产生影响。

## 可操作的高级指导 { #guidance }

这对Web开发人员意味着什么？解析和编译成本是**不再那么慢**正如我们曾经认为的那样。对于 JavaScript 捆绑包，需要关注的三件事是：

*   **缩短下载时间**
    *   保持你的 JavaScript 捆绑包较小，特别是对于移动设备。小捆绑包可提高下载速度、降低内存使用率并降低 CPU 成本。
    *   避免只有一个大的捆绑包;如果一个束超过 ~50–100 kB，则将其拆分为单独的较小束。（使用 HTTP/2 多路复用，可以同时传输多个请求和响应消息，从而减少其他请求的开销。
    *   在移动设备上，您希望少得多的运费，特别是由于网络速度，但也要保持较低的普通内存使用率。
*   **缩短执行时间**
    *   避免[长期任务](https://w3c.github.io/longtasks/)这可以使主线程保持忙碌，并且可以推出页面交互的速度。下载后，脚本执行时间现在是主要成本。
*   **避免使用大型内联脚本**（因为它们仍然在主线程上解析和编译）。一个好的经验法则是：如果脚本超过1 kB，请避免内联它（也因为1 kB是当[代码缓存](/blog/code-caching-for-devs)启动外部脚本）。

## 为什么下载和执行时间很重要？{ #download-执行 }

为什么优化下载和执行时间很重要？下载时间对于低端网络至关重要。尽管全球4G（甚至5G）的增长，我们的[有效连接类型](https://developer.mozilla.org/en-US/docs/Web/API/NetworkInformation/effectiveType)仍然不一致，我们中的许多人在旅途中遇到感觉像3G（或更糟）的速度。

JavaScript执行时间对于CPU速度较慢的手机非常重要。由于CPU、GPU、热节流等方面的差异，高端手机和低端手机的性能存在巨大差异。这对 JavaScript 的性能很重要，因为执行是 CPU 密集型的。

事实上，在页面在Chrome等浏览器中加载的总时间中，任何地方都可以花费高达30%的时间用于JavaScript执行。下面是从高端桌面计算机上具有非常典型工作负载（Reddit.com）的站点加载的页面：

![JavaScript processing represents 10–30% of time spent in V8 during page load.](../_img/cost-of-javascript-2019/reddit-js-processing.svg)

在移动设备上，与高端设备（Pixel 3）相比，中值手机（Moto G4）执行Reddit的JavaScript需要3-4×的时间，而在低端设备上（<100美元阿尔卡特1X）则需要超过6×

![The cost of Reddit’s JavaScript across a few different device classes (low-end, average, and high-end)](../_img/cost-of-javascript-2019/reddit-js-processing-devices.svg)

：：：备注
**注意：**Reddit在桌面和移动网络方面具有不同的体验，因此MacBook Pro的结果无法与其他结果进行比较。
:::

当你试图优化JavaScript执行时间时，请留意[长期任务](https://web.dev/long-tasks-devtools/)这可能会长时间独占UI线程。这些可能会阻止关键任务的执行，即使页面在视觉上看起来已准备就绪。将它们分解成更小的任务。通过拆分代码并确定其加载顺序的优先级，可以更快地使页面交互，并希望具有更低的输入延迟。

![Long tasks monopolize the main thread. You should break them up.](../_img/cost-of-javascript-2019/long-tasks.png)

## V8 在改进解析/编译方面做了哪些工作？{ #v8改进 }

自Chrome 60以来，V8中的原始JavaScript解析速度提高了2×与此同时，由于Chrome中的其他优化工作并行化了它，原始解析（和编译）成本变得不那么明显/重要。

V8通过在工作线程上进行解析和编译，将主线程上的解析和编译工作量平均减少了40%（例如，Facebook上为46%，Pinterest上为62%），最高改进为81%（YouTube）。这是对现有的非主线程流式解析/编译的补充。

![V8 parse times across different versions](../_img/cost-of-javascript-2019/chrome-js-parse-times.svg)

我们还可以可视化这些变化对 Chrome 版本中不同版本的 V8 的 CPU 时间影响。在Chrome 61解析Facebook的JS所花费的时间相同的情况下，Chrome 75现在可以解析Facebook的JS和Twitter的JS的6倍。

![In the time it took Chrome 61 to parse Facebook’s JS, Chrome 75 can now parse both Facebook’s JS and 6 times Twitter’s JS.](../_img/cost-of-javascript-2019/js-parse-times-websites.svg)

让我们深入了解这些更改是如何解锁的。简而言之，脚本资源可以在工作线程上进行流式解析和编译，这意味着：

*   V8 可以在不阻塞主线程的情况下解析并编译 JavaScript。
*   一旦完整的 HTML 解析器遇到`<script>`标记。对于解析器阻塞脚本，HTML 解析器产生，而对于异步脚本，它继续。
*   对于大多数实际连接速度，V8 的解析速度比下载快，因此 V8 在最后一个脚本字节下载后几毫秒内完成解析和编译。

不那么简短的解释是...旧版本的Chrome会在开始解析脚本之前完整下载脚本，这是一种简单的方法，但它没有充分利用CPU。在版本 41 和 68 之间，Chrome 在下载开始后立即开始在单独的线程上解析异步和延迟脚本。

![Scripts arrive in multiple chunks. V8 starts streaming once it’s seen at least 30 kB.](../_img/cost-of-javascript-2019/script-streaming-1.svg)

在Chrome 71中，我们转向了基于任务的设置，其中调度程序可以一次解析多个异步/延迟脚本。此更改的影响是主线程解析时间缩短了约 20%，在实际网站上测量的 TTI/FID 总体提高了约 2%。

![Chrome 71 moved to a task-based setup where the scheduler could parse multiple async/deferred scripts at once.](../_img/cost-of-javascript-2019/script-streaming-2.svg)

在Chrome 72中，我们改用流媒体作为解析的主要方式：现在也以这种方式解析常规同步脚本（尽管不是内联脚本）。如果主线程需要，我们也停止取消基于任务的解析，因为这只会不必要地重复任何已经完成的工作。

[以前版本的 Chrome](/blog/v8-release-75#script-streaming-directly-from-network)支持流式解析和编译，其中来自网络的脚本源数据必须先进入Chrome的主线程，然后才能转发到流送器。

这通常会导致流式解析器等待已经从网络到达的数据，但尚未转发到流式处理任务，因为它被主线程上的其他工作（如HTML解析，布局或JavaScript执行）阻止。

我们现在正在尝试在预加载时开始解析，而主线程反弹事先是一个阻碍因素。

Leszek Swirski的BlinkOn演示文稿详细介绍了以下内容：

<figure>
  <div class="video video-16:9">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/D1UJgiG4_NI" allow="picture-in-picture" allowfullscreen loading="lazy"></iframe>
  </div>
  <figcaption><a href="https://www.youtube.com/watch?v=D1UJgiG4_NI">“Parsing JavaScript in zero* time”</a> as presented by Leszek Swirski at BlinkOn 10.</figcaption>
</figure>

## 这些更改如何反映您在 DevTools 中看到的内容？

除上述内容外，还有[DevTools 中的一个问题](https://bugs.chromium.org/p/chromium/issues/detail?id=939275)以一种暗示它正在使用CPU（完整块）的方式呈现整个解析器任务。但是，每当解析器缺少数据（需要绕过主线程）时，它就会阻止它。由于我们从单个流式处理线程转移到流式处理任务，因此这变得非常明显。以下是您在 Chrome 69 中看到的内容：

![The DevTools issue that rendered the entire parser task in a way that hints that it’s using CPU (full block)](../_img/cost-of-javascript-2019/devtools-69.png)

“解析脚本”任务显示为需要 1.08 秒。但是，解析JavaScript并不是那么慢！除了等待数据通过主线程之外，大部分时间都花在了什么事情上。

Chrome 76描绘了一幅不同的画面：

![In Chrome 76, parsing is broken up into multiple smaller streaming tasks.](../_img/cost-of-javascript-2019/devtools-76.png)

通常，DevTools 性能窗格非常适合获取页面上发生的情况的高级概述。有关特定于 V8 的详细指标，例如 JavaScript 解析和编译时间，我们建议[将 Chrome 跟踪与运行时调用统计信息 （RCS） 结合使用](/docs/rcs).在 RCS 结果中，`Parse-Background`和`Compile-Background`告诉您在主线程上解析和编译 JavaScript 所花费的时间，而`Parse`和`Compile`捕获主线程指标。

![](../_img/cost-of-javascript-2019/rcs.png)

## 这些变化对现实世界的影响是什么？{ #impact }

让我们看一下真实网站的一些示例以及脚本流如何应用。

![Main thread vs. worker thread time spent parsing and compiling Reddit’s JS on a MacBook Pro](../_img/cost-of-javascript-2019/reddit-main-thread.svg)

Reddit.com 有几个 100 kB+ 的捆绑包，这些捆绑包被包裹在外部函数中，导致大量[惰性编译](/blog/preparser)在主线程上。在上面的图表中，主线程时间才是真正重要的，因为保持主线程繁忙会延迟交互性。Reddit将大部分时间花在主线程上，而Worker/后台线程的使用最少。

他们将受益于将一些较大的捆绑包拆分为较小的捆绑包（例如每个捆绑包50 kB），而无需包装以最大化并行化 - 这样每个捆绑包都可以单独进行流解析+编译，并减少启动期间的主线程解析/编译。

![Main thread vs. worker thread time spent parsing and compiling Facebook’s JS on a MacBook Pro](../_img/cost-of-javascript-2019/facebook-main-thread.svg)

我们也可以查看像 Facebook.com 这样的网站。Facebook在大约292个请求中加载了大约6MB的压缩JS，其中一些是异步的，一些是预加载的，还有一些是以较低的优先级获取的。他们的许多脚本都非常小且细粒度 - 这可以帮助后台/工作线程上进行整体并行化，因为这些较小的脚本可以同时进行流式解析/编译。

请注意，您可能不是Facebook，并且可能没有像Facebook或Gmail这样长期存在的应用程序，在这些应用程序中，如此多的脚本在桌面上可能是合理的。但是，一般来说，请保持捆绑包的粗糙，只加载您需要的东西。

尽管大多数 JavaScript 解析和编译工作可以在后台线程上以流方式进行，但某些工作仍然必须在主线程上进行。当主线程繁忙时，页面无法响应用户输入。请密切关注下载和执行代码对UX的影响。

：：：备注
**注意：**目前，并非所有 JavaScript 引擎和浏览器都实现脚本流作为加载优化。我们仍然相信这里的整体指导会带来全面的用户体验。
:::

## 解析 JSON 的成本 { #json }

因为JSON语法比JavaScript的语法简单得多，所以JSON可以比JavaScript更有效地解析。这些知识可用于提高交付大型类似 JSON 的配置对象文本（如内联 Redux 存储）的 Web 应用的启动性能。而不是将数据内联为 JavaScript 对象文本，如下所示：

```js
const data = { foo: 42, bar: 1337 }; // 🐌
```

...它可以以JSON字符串化形式表示，然后在运行时进行JSON解析：

```js
const data = JSON.parse('{"foo":42,"bar":1337}'); // 🚀
```

只要 JSON 字符串只计算一次，则`JSON.parse`方法是[快得多](https://github.com/GoogleChromeLabs/json-parse-benchmark)与JavaScript对象文本相比，特别是对于冷负载。一个好的经验法则是将此技术应用于10 kB或更大的对象 - 但与性能建议一样，在进行任何更改之前测量实际影响。

![JSON.parse('…') is much faster to parse, compile, and execute compared to an equivalent JavaScript literal — not just in V8 (1.7× as fast), but in all major JavaScript engines.](../_img/cost-of-javascript-2019/json.svg)

以下视频更详细地介绍了性能差异的来源，从 02：10 标记开始。

<figure>
  <div class="video video-16:9">
    <iframe width="560" height="315" src="https://www.youtube.com/embed/ff4fgQxPaO0?start=130" allow="picture-in-picture" allowfullscreen loading="lazy"></iframe>
  </div>
  <figcaption><a href="https://www.youtube.com/watch?v=ff4fgQxPaO0">“Faster apps with <code>JSON.parse</code>”</a> as presented by Mathias Bynens at #ChromeDevSummit 2019.</figcaption>
</figure>

看[我们*JSON ⊂ ECMAScript*功能说明](/features/subsume-json#embedding-json-parse)对于一个示例实现，给定一个任意对象，生成一个有效的JavaScript程序，`JSON.parse`是它。

对大量数据使用纯对象文本时，还存在一个额外的风险：它们可能会被解析*两次*!

1.  第一次传递发生在文本准备就绪时。
2.  第二次传递发生在文本被延迟解析时。

第一次通过是无法避免的。幸运的是，通过将对象文本放在顶层或[外籍家常便器](/blog/preparser#pife).

## 在重复访问时解析/编译怎么样？{ #repeat访问 }

V8 的（字节）代码缓存优化可以提供帮助。当第一次请求脚本时，Chrome 会下载该脚本并将其提供给 V8 进行编译。它还将文件存储在浏览器的磁盘缓存中。当第二次请求JS文件时，Chrome会从浏览器缓存中获取该文件，并再次将其提供给V8进行编译。但是，这一次，编译的代码将序列化，并作为元数据附加到缓存的脚本文件。

![Visualization of how code caching works in V8](../_img/cost-of-javascript-2019/code-caching.png){ .no-darkening }

第三次，Chrome 从缓存中获取文件和文件的元数据，并将两者都交给 V8。V8 反序列化元数据，可以跳过编译。如果前两次访问发生在 72 小时内，代码缓存就会启动。如果使用服务工作线程缓存脚本，Chrome 还具有预先的代码缓存功能。您可以在 中阅读有关代码缓存的详细信息[面向 Web 开发人员的代码缓存](/blog/code-caching-for-devs).

## 结论

下载和执行时间是2019年加载脚本的主要瓶颈。为首屏内容提供一小包同步（内联）脚本，并为页面的其余部分提供一个或多个延迟脚本。分解您的大型捆绑包，以便您只专注于用户在需要时需要的运输代码。这最大限度地提高了 V8 中的并行化。

在移动设备上，由于网络、内存消耗和较慢 CPU 的执行时间，您希望交付的脚本要少得多。平衡延迟与可缓存性，以最大限度地提高主线程外可能发生的解析和编译工作量。

## 进一步阅读

*   [极快的解析，第 1 部分：优化扫描程序](/blog/scanner)
*   [极快解析，第 2 部分：惰性解析](/blog/preparser)
