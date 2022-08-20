***

标题： “免费垃圾收集”
作者：“Hannes Payer和Ross McIlroy，Idle Garbage Collectors”
化身：

*   “hannes-payer”
*   “罗斯-麦克罗伊”
    日期： 2015-08-07 13：33：37
    标签：
*   内部
*   记忆
    描述： “Chrome 41将昂贵的内存管理操作隐藏在小块的，否则未使用的空闲时间块中，从而减少了卡顿。

***

JavaScript性能仍然是Chrome价值观的关键方面之一，尤其是在实现流畅体验方面。从Chrome 41开始，V8利用一种新技术，通过将昂贵的内存管理操作隐藏在小的，否则未使用的空闲时间块中来提高Web应用程序的响应能力。因此，Web开发人员应该期望更流畅的滚动和黄油动画，并且由于垃圾回收而大大减少卡顿。

许多现代语言引擎，如Chrome的V8 JavaScript引擎，可以动态管理运行应用程序的内存，这样开发人员就不需要自己担心。引擎定期传递分配给应用程序的内存，确定不再需要哪些数据，并将其清除以释放空间。此过程称为[垃圾回收](https://en.wikipedia.org/wiki/Garbage_collection_\(computer_science\)).

在 Chrome 中，我们努力提供流畅的每秒 60 帧 （FPS） 视觉体验。尽管 V8 已经尝试以小块的形式执行垃圾回收，但较大的垃圾回收操作可以并且确实在不可预测的时间发生 - 有时是在动画的中间 - 暂停执行并阻止 Chrome 达到 60 FPS 的目标。

Chrome 41 包括一个[眨眼渲染引擎的任务计划程序](https://blog.chromium.org/2015/04/scheduling-tasks-intelligently-for\_30.html)这样可以确定延迟敏感任务的优先级，以确保Chrome保持响应迅速。除了能够确定工作的优先级外，该任务调度程序还集中了解系统的繁忙程度，需要执行的任务以及每个任务的紧急程度。因此，它可以估计Chrome何时可能处于空闲状态，以及预计保持闲置的时间。

例如，当 Chrome 在网页上显示动画时。动画将以60 FPS的速度更新屏幕，为Chrome提供大约16.6毫秒的时间来执行更新。因此，Chrome 将在显示前一帧后立即开始处理当前帧，并为此新帧执行输入、动画和帧渲染任务。如果Chrome在不到16.6毫秒的时间内完成了所有这些工作，那么在需要开始渲染下一帧之前，它在剩余时间内没有其他事情可做。Chrome的调度程序使V8能够利用这一点*空闲时间段*通过调度特殊*空闲任务*当Chrome处于空闲状态时。

![Figure 1: Frame rendering with idle tasks](/\_img/free-garbage-collection/frame-rendering.png)

空闲任务是特殊的低优先级任务，当计划程序确定它处于空闲状态时运行。空闲任务被赋予一个截止时间，这是调度程序对它预计保持空闲时间的估计。在图 1 的动画示例中，这将是开始绘制下一帧的时间。在其他情况下（例如，当屏幕上没有发生任何活动时），这可能是计划运行下一个待处理任务的时间，上限为50毫秒，以确保Chrome对意外的用户输入保持响应。空闲任务使用截止时间来估计它可以执行多少工作，而不会导致输入响应卡顿或延迟。

在空闲任务中完成的垃圾回收对关键的延迟敏感操作是隐藏的。这意味着这些垃圾回收任务是“免费”完成的。为了理解 V8 是如何做到这一点的，值得回顾一下 V8 当前的垃圾回收策略。

## 深入了解 V8 的垃圾收集引擎

V8 使用[代际垃圾回收器](http://www.memorymanagement.org/glossary/g.html#term-generational-garbage-collection)Javascript堆分为一个小的年轻一代用于新分配的对象和一个大的老一代用于长寿对象。[因为大多数物体在年轻时就死了](http://www.memorymanagement.org/glossary/g.html#term-generational-hypothesis)，这种代际策略使垃圾回收器能够在较小的年轻一代（称为拾荒者）中执行定期的，简短的垃圾回收，而不必跟踪旧一代中的对象。

年轻一代使用[半空间](http://www.memorymanagement.org/glossary/s.html#semi.space)分配策略，其中新对象最初在年轻一代的活跃半空间中分配。一旦该半空间变满，清除操作就会将活动对象移动到另一个半空间。已经移动过一次的物体被提升到老一代，被认为是长寿的。移动活动对象后，新的半空间将变为活动状态，旧半空间中任何剩余的死对象都将被丢弃。

因此，年轻一代拾荒的持续时间取决于年轻一代中活体物体的大小。当大多数对象在年轻一代中变得无法访问时，清除将很快（<1毫秒）。但是，如果大多数物体在清除中幸存下来，则清除的持续时间可能会明显更长。

当旧一代中活动对象的大小超出启发式派生的限制时，将执行整个堆的主要集合。老一代使用[标记和扫描收集器](http://www.memorymanagement.org/glossary/m.html#term-mark-sweep)进行了多项优化，以改善延迟和内存消耗。标记延迟取决于必须标记的活动对象的数量，对于大型 Web 应用程序，整个堆的标记可能需要 100 毫秒以上。为了避免主线程暂停这么长时间，V8早就具备了这样的能力：[以许多小步骤增量标记活动对象](https://blog.chromium.org/2011/11/game-changer-for-interactive.html)，目的是使每个标记步骤的持续时间保持在5毫秒以下。

标记后，通过扫描整个旧一代内存，可用内存再次可供应用程序使用。此任务由专用的清扫器线程同时执行。最后，执行内存压缩以减少旧一代中的内存碎片。此任务可能非常耗时，并且仅在内存碎片出现问题时才执行。

总之，有四个主要的垃圾回收任务：

1.  年轻一代的拾荒者，通常速度很快
2.  增量标记执行的标记步骤，根据步长，可以任意长
3.  完整的垃圾回收，这可能需要很长时间
4.  具有主动内存压缩的完全垃圾回收，这可能需要很长时间，但会清理碎片内存

为了在空闲期间执行这些操作，V8 将垃圾回收空闲任务发布到调度程序。当这些空闲任务运行时，它们将获得一个应完成的截止时间。V8 的垃圾回收空闲时间处理程序评估应执行哪些垃圾回收任务以减少内存消耗，同时遵守截止时间以避免将来帧渲染中的卡顿或输入延迟。

如果应用程序的测量分配率显示年轻一代在下一个预期空闲期之前可能已满，则垃圾回收器将在空闲任务期间执行年轻一代的清理。此外，它还计算最近清除任务所需的平均时间，以预测未来清理的持续时间，并确保它不会违反空闲任务截止日期。

当旧一代中活动对象的大小接近堆限制时，将启动增量标记。增量标记步骤可以按应标记的字节数线性缩放。根据测量的平均标记速度，垃圾回收空闲时间处理程序会尝试将尽可能多的标记工作放入给定的空闲任务中。

如果旧一代几乎已满，并且提供给任务的截止时间估计足以完成收集，则在空闲任务期间计划完全垃圾回收。收集暂停时间是根据标记速度乘以分配的对象数来预测的。仅当网页长时间处于空闲状态时，才会执行具有额外压缩的完全垃圾回收。

## 绩效评估

为了评估在空闲时间运行垃圾回收的影响，我们使用了Chrome的[遥测性能基准框架](https://www.chromium.org/developers/telemetry)以评估热门网站在加载时滚动的流畅程度。我们对[前 25 名](https://code.google.com/p/chromium/codesearch#chromium/src/tools/perf/benchmarks/smoothness.py\&l=15)Linux 工作站上的站点以及[典型的移动网站](https://code.google.com/p/chromium/codesearch#chromium/src/tools/perf/benchmarks/smoothness.py\&l=104)在Android Nexus 6智能手机上，两者都打开流行的网页（包括Gmail，Google Docs和YouTube等复杂的网络应用程序）并滚动其内容几秒钟。Chrome旨在以60 FPS的速度继续滚动，以获得流畅的用户体验。

图 2 显示了在空闲时间安排的垃圾回收的百分比。与Nexus 6相比，工作站更快的硬件导致更多的整体空闲时间，从而可以在此空闲时间安排更大比例的垃圾收集（43%，而Nexus 6上为31%），从而使我们的垃圾回收率提高了约7%[卡顿指标](https://www.chromium.org/developers/design-documents/rendering-benchmarks).

![Figure 2: The percentage of garbage collection that occurs during idle time](/\_img/free-garbage-collection/idle-time-gc.png)

除了提高页面呈现的平滑度外，这些空闲期间还提供了在页面完全空闲时执行更主动的垃圾回收的机会。Chrome 45 最近的改进利用了这一点，大大减少了空闲前台标签所消耗的内存量。图 3 显示了与 Chrome 43 中的同一页面相比，Gmail 的 JavaScript 堆在空闲时的内存使用量如何减少约 45%。

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/ij-AFUfqFdI" width="640" height="360" loading="lazy"></iframe>
  </div>
  <figcaption>Figure 3: Memory usage for Gmail on latest version of Chrome 45 (left) vs. Chrome 43</figcaption>
</figure>

这些改进表明，通过更智能地了解何时执行成本高昂的垃圾回收操作，可以隐藏垃圾回收暂停。Web开发人员不再需要担心垃圾回收暂停，即使目标是丝般流畅的60 FPS动画。请继续关注更多改进，因为我们推动了垃圾回收计划的界限。
