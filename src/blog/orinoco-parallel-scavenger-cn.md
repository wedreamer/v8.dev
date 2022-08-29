***

标题： '奥里诺科：年轻一代垃圾收集'
作者：'Ulan Degenbaev，Michael Lippautz和Hannes Payer，朋友[赞](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual)'
化身：

*   '乌兰-德根巴耶夫'
*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2017-11-29 13：33：37
    标签：
*   内部
*   记忆
    描述： “本文介绍了并行清除器，这是 V8 主要并发和并行垃圾收集器 Orinoco 的最新功能之一。

***

V8 中的 JavaScript 对象分配在由 V8 的垃圾回收器管理的堆上。在之前的博客文章中，我们已经讨论了我们如何[减少垃圾回收暂停时间](/blog/jank-busters)([不止一次](/blog/orinoco)） 和[内存消耗](/blog/optimizing-v8-memory).在这篇博客文章中，我们介绍了并行清道夫，这是Orinoco的最新功能之一，V8主要是并发和并行垃圾收集器，并讨论了我们在过程中实现的设计决策和替代方法。

V8 将其托管堆划分为几代，其中对象最初在年轻一代的“托儿所”中分配。在垃圾回收中幸存下来后，对象被复制到中间一代，这仍然是年轻一代的一部分。在另一个垃圾回收中幸存下来后，这些对象被移动到旧一代中（参见图 1）。V8 实现了两个垃圾回收器：一个经常收集年轻一代，另一个收集包括年轻一代和老一代在内的完整堆。从老到年轻一代的参考是年轻一代垃圾收集的根源。这些引用是[记录](/blog/orinoco)在移动对象时提供高效的根标识和引用更新。

![Figure 1: Generational garbage collection](../_img/orinoco-parallel-scavenger/generational-gc.png)

由于年轻一代相对较小（V8中高达16MiB），因此它很快就会被物体填满，并且需要频繁收集。在M62之前，V8使用切尼半空间复制垃圾收集器（见下文），将年轻一代分成两半。在JavaScript执行期间，只有一半的年轻一代可用于分配对象，而另一半则保持空白。在年轻的垃圾回收期间，活动对象从一半复制到另一半，从而动态压缩内存。已复制过一次的活动对象被视为中间一代的一部分，并提升为旧一代。

**从 v6.2 开始，V8 将收集年轻一代的默认算法切换到并行清道夫**似[霍尔斯特德的半空间复制收集器](https://dl.acm.org/citation.cfm?id=802017)不同之处在于，V8 使用动态而不是静态工作跨多个线程窃取。在下面，我们解释了三种算法：a）单线程切尼半空间复制收集器，b）并行Mark-Evacuate方案，以及c）并行清道夫。

## 单线程切尼半空间复制

在 v6.2 之前，使用 V8[切尼的半空间复制算法](https://dl.acm.org/citation.cfm?doid=362790.362798)它非常适合单核执行和分代方案。在年轻一代集合之前，两个半空间的内存都被提交并分配了适当的标签：包含当前对象集的页面称为*从太空*而对象复制到的页面被调用*到太空*.

清道夫将调用堆栈中的引用以及从老一代到年轻一代的引用视为根。图 2 说明了最初拾荒者扫描这些根目录并复制可在*从太空*尚未复制到*到太空*.已在垃圾回收中幸存下来的对象将提升（移动）到旧一代。在根扫描和第一轮复制之后，将扫描新分配到空间中的对象以查找参考。同样，扫描所有升级的对象以查找*从太空*.这三个阶段在主线程上交错。该算法将继续，直到无法从任一位置访问更多新对象为止*到太空*或老一辈。此时*从太空*仅包含无法访问的对象，即它仅包含垃圾。

![Figure 2: Cheney’s semispace copying algorithm used for young generation garbage collections in V8](../_img/orinoco-parallel-scavenger/cheneys-semispace-copy.png)

![Processing](../_img/orinoco-parallel-scavenger/cheneys-semispace-copy-processing.png)

## 平行标记-撤离

我们试验了基于V8的完整Mark-Sweep-Compact收集器的并行Mark-Evacuate算法。主要优点是利用完整的 Mark-Sweep-Compact 收集器中已经存在的垃圾回收基础结构。该算法由三个阶段组成：标记、复制和更新指针，如图 3 所示。为了避免年轻一代的页面扫荡以维护自由列表，年轻一代仍然使用半空间进行维护，该半空间始终通过将活动对象复制到其中来保持紧凑*到太空*在垃圾回收期间。年轻一代最初是平行的。标记后，活动对象将并行复制到其相应的空间。工作基于逻辑页进行分配。参与复制的线程保留自己的本地分配缓冲区 （LAG），这些缓冲区在完成复制时合并。复制后，将应用相同的并行化方案来更新对象间指针。这三个阶段是同步执行的，即，虽然阶段本身是并行执行的，但线程必须在继续下一阶段之前进行同步。

![Figure 3: Young Generation Parallel Mark-Evacuate garbage collection in V8](../_img/orinoco-parallel-scavenger/parallel-mark-evacuate.png)

![Processing](../_img/orinoco-parallel-scavenger/parallel-mark-evacuate-processing.png)

## 平行拾荒

并行 Mark-Evacuate 收集器将计算活动性、复制活动对象和更新指针的各个阶段分开。一个明显的优化是合并这些阶段，从而产生一种同时标记、复制和更新指针的算法。通过合并这些阶段，我们实际上得到了V8使用的并行清道夫，这是一个类似于[霍尔斯特德](https://dl.acm.org/citation.cfm?id=802017)半空间收集器的不同之处在于 V8 使用动态工作窃取和简单的负载平衡机制来扫描根目录（参见图 4）。与单线程切尼算法一样，这些阶段是：扫描根源，在年轻一代中复制，晋升到老一代，以及更新指针。我们发现，根集的大部分通常是从老一代到年轻一代的引用。在我们的实现中，记住集是按页维护的，这自然会在垃圾回收线程之间分配根集。然后并行处理对象。新找到的对象将添加到全局工作列表中，垃圾回收线程可以从中窃取。此工作列表提供快速任务本地存储以及用于共享工作的全局存储。屏障可确保当当前处理的子图不适合工作窃取（例如，线性对象链）时，任务不会过早终止。所有阶段在每个任务上并行和交错执行，从而最大限度地提高了工作线程任务的利用率。

![Figure 4: Young generation parallel Scavenger in V8](../_img/orinoco-parallel-scavenger/parallel-scavenge.png)

![Processing](../_img/orinoco-parallel-scavenger/parallel-scavenge-processing.png)

## 结果和成果

清道夫算法最初在设计时考虑了最佳的单核性能。从那时起，世界发生了变化。CPU内核通常很充足，即使在低端移动设备上也是如此。更重要的是，[经常](https://dl.acm.org/citation.cfm?id=2968469)这些内核实际上已启动并运行。为了充分利用这些内核，V8 垃圾回收器的最后一个顺序组件之一 Scavenger 必须进行现代化改造。

并行 Mark-Evacuate 收集器的最大优点是可以获得精确的活动信息。例如，此信息可用于通过移动和重新链接包含大部分活动对象的页面来避免复制，这也由完整的Mark-Sweep-Compact收集器执行。然而，在实践中，这主要是在合成基准测试上观察到的，很少出现在真实的网站上。并行 Mark-Evacuate 收集器的缺点是执行三个单独的锁步阶段的开销。当在具有大部分死对象的堆上调用垃圾回收器时，这种开销尤其明显，许多实际网页就是这种情况。请注意，在包含大多数死对象的堆上调用垃圾回收实际上是理想的方案，因为垃圾回收通常受活动对象大小的限制。

并行 Scavenger 通过在小型或几乎为空的堆上提供接近优化的 Cheney 算法的性能来缩小这种性能差距，同时在堆变大并包含大量活动对象的情况下仍提供高吞吐量。

在许多其他平台中，V8 支持[胳膊很大。小](https://developer.arm.com/technologies/big-little).虽然卸载小内核上的工作有利于电池寿命，但当小内核的工作包太大时，它可能导致主线程停滞。我们观察到，页面级并行性不一定在大上实现负载平衡工作。由于页面数量有限，年轻一代的垃圾收集很少。清道夫通过使用显式工作列表和工作窃取提供中等粒度同步，自然地解决了这个问题。

![Figure 5: Total young generation garbage collection time (in ms) across various websites](../_img/orinoco-parallel-scavenger/results.png)

V8现在与平行的清道夫一起提供**将主线年轻一代垃圾收集总时间减少约20%-50%**跨大量基准测试（[关于我们性能瀑布的详细信息](https://chromeperf.appspot.com/group_report?rev=489898)).图 5 显示了各种真实网站实现的比较，显示了以下方面的改进**55% （2×）**.在保持最小暂停时间的同时，可以在最大和平均暂停时间上观察到类似的改进。并行 Mark-Evacuate 收集器方案仍具有优化潜力。如果您想了解接下来会发生什么，请继续关注。
