***

标题： “优化 V8 内存消耗”
作者：“V8内存卫生工程师Ulan Degenbaev，Michael Lippautz，Hannes Payer和Toon Verwaest”
化身：

*   '乌兰-德根巴耶夫'
*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2016-10-07 13：33：37
    标签：
*   记忆
*   基准
    描述： “V8 团队分析并显著减少了几个网站的内存占用，这些网站被确定为现代 Web 开发模式的代表。

***

内存消耗是 JavaScript 虚拟机性能权衡空间中的一个重要维度。在过去的几个月里，V8团队分析并显着减少了几个网站的内存占用，这些网站被确定为现代Web开发模式的代表。在这篇博客文章中，我们介绍了我们在分析中使用的工作负载和工具，概述了垃圾回收器中的内存优化，并展示了我们如何减少 V8 解析器及其编译器消耗的内存。

## 基准

为了分析 V8 并发现对最大用户数量有影响的优化，定义可重现、有意义的工作负载并模拟常见的真实世界 JavaScript 使用场景至关重要。完成此任务的一个很好的工具是[遥测](https://catapult.gsrc.io/telemetry)，这是一个性能测试框架，可在 Chrome 中运行脚本化网站交互并记录所有服务器响应，以便在我们的测试环境中对这些交互进行可预测的重播。我们选择了一组热门新闻、社交和媒体网站，并为它们定义了以下常见的用户交互：

浏览新闻和社交网站的工作量：

1.  打开热门新闻或社交网站，例如黑客新闻。
2.  单击第一个链接。
3.  等到新网站加载完毕。
4.  向下滚动几页。
5.  单击后退按钮。
6.  单击原始网站上的下一个链接，然后重复步骤3-6几次。

浏览媒体网站的工作量：

1.  在热门媒体网站上打开项目，例如 YouTube 上的视频。
2.  通过等待几秒钟来消耗该项目。
3.  单击下一项并重复步骤 2–3 几次。

捕获工作流后，可以根据需要针对Chrome的开发版本经常重放它，例如每次有新版本的V8时。在播放期间，V8 的内存使用情况以固定的时间间隔进行采样，以获得有意义的平均值。可以找到基准[这里](https://cs.chromium.org/chromium/src/tools/perf/page_sets/system_health/browsing_stories.py?q=browsing+news\&sq=package:chromium\&dr=CS\&l=11).

## 内存可视化

通常，优化性能时的主要挑战之一是清楚地了解内部 VM 状态，以跟踪进度或权衡潜在的权衡。为了优化内存消耗，这意味着在执行期间准确跟踪 V8 的内存消耗。必须跟踪两类内存：分配给 V8 托管堆的内存和分配给C++堆的内存。这**V8 堆统计信息**功能是开发人员在 V8 内部结构上使用的一种机制，用于深入了解两者。当`--trace-gc-object-stats`标志是在运行 Chrome（54 或更高版本）时指定的，或者`d8`命令行界面，V8 将与内存相关的统计信息转储到控制台。我们构建了一个自定义工具，[V8 堆可视化工具](https://mlippautz.github.io/v8-heap-stats/)，以可视化此输出。该工具为托管堆和C++堆显示基于时间线的视图。该工具还提供了某些内部数据类型的内存使用情况的详细细分，以及每种类型基于大小的直方图。

优化过程中的常见工作流涉及选择在时间轴视图中占据大部分堆的实例类型，如图 1 所示。选择实例类型后，该工具将显示此类型使用情况的分布。在此示例中，我们选择了 V8 的内部 FixedArray 数据结构，这是一个非类型化的类似矢量的容器，在 VM 的各种位置无处不在。图 2 显示了一个典型的 FixedArray 分布，我们可以看到大部分内存可以归因于特定的 FixedArray 使用场景。在这种情况下，FixedArrays被用作稀疏JavaScript数组（我们称之为DICTIONARY_ELEMENTS）的后备存储。有了这些信息，就可以回溯到实际代码，并验证此分布是否确实是预期行为，或者是否存在优化机会。我们使用该工具来识别许多内部类型的低效率。

![Figure 1: Timeline view of managed heap and off-heap memory](../_img/optimizing-v8-memory/timeline-view.png)

![Figure 2: Distribution of instance type](../_img/optimizing-v8-memory/distribution.png)

图 3 显示了C++堆内存消耗，主要由区域内存（V8 使用的临时内存区域用于短时间;下面将更详细地讨论）组成。 由于 V8 解析器和编译器最广泛地使用区域内存，因此峰值对应于解析和编译事件。行为良好的执行仅由峰值组成，表示一旦不再需要内存，就会释放内存。相反，平台（即较长的时间段和较高的内存消耗）表示存在优化空间。

![Figure 3: Zone memory](../_img/optimizing-v8-memory/zone-memory.png)

早期采用者也可以尝试集成到[Chrome 的跟踪基础架构](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool).因此，您需要运行最新的Chrome Canary`--track-gc-object-stats`和[捕获跟踪](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/recording-tracing-runs#TOC-Capture-a-trace-on-Chrome-desktop)包括类别`v8.gc_stats`.然后，数据将显示在`V8.GC_Object_Stats`事件。

## JavaScript 堆大小减小

垃圾回收吞吐量、延迟和内存消耗之间存在固有的权衡。例如，通过使用更多内存来避免频繁的垃圾回收调用，可以减少垃圾回收延迟（这会导致用户可见的卡顿）。对于低内存移动设备，即 RAM 小于 512 MB 的设备，优先考虑延迟和吞吐量而不是内存消耗可能会导致 Android 上的内存不足崩溃和标签页挂起。

为了更好地平衡这些低内存移动设备的正确权衡，我们引入了一种特殊的内存缩减模式，该模式可以调整几个垃圾回收启发式方法，以降低 JavaScript 垃圾回收堆的内存使用率。

1.  在完全垃圾回收结束时，V8 的堆增长策略会根据具有一些额外松弛度的活动对象的数量来确定下一次垃圾回收何时发生。在内存缩减模式下，V8 使用较少的松弛，从而由于垃圾回收更频繁而导致内存使用量减少。
2.  此外，此估计值被视为硬性限制，迫使未完成的增量标记工作在主垃圾回收暂停中完成。通常，当不在内存缩减模式下时，未完成的增量标记工作可能会导致任意超过此限制，仅在标记完成后触发主垃圾回收暂停。
3.  通过执行更主动的内存压缩，可以进一步减少内存碎片。

图 4 描述了自 Chrome 53 以来在低内存设备上的一些改进。最值得注意的是，移动版《纽约时报》基准测试的平均 V8 堆内存消耗减少了约 66%。总体而言，我们观察到这组基准测试的平均 V8 堆大小减少了 50%。

![Figure 4: V8 heap memory reduction since Chrome 53 on low-memory devices](../_img/optimizing-v8-memory/heap-memory-reduction.png)

最近引入的另一项优化不仅减少了低内存设备上的内存，而且减少了更强大的移动和台式机。将 V8 堆页面大小从 1 MB 减少到 512 kB 可在活动对象不多时减少内存占用，并将整体内存碎片降低多达 2×。它还允许 V8 执行更多的压缩工作，因为较小的工作块允许内存压缩线程并行完成更多的工作。

## 区域内存减少

除了 JavaScript 堆之外，V8 还使用堆外内存进行内部 VM 操作。最大的内存块通过称为*区*.区域是一种基于区域的内存分配器，它支持快速分配和批量解除分配，其中所有区域分配的内存在区域被销毁时立即释放。区域在整个 V8 的解析器和编译器中使用。

Chrome 55的主要改进之一是减少了后台解析期间的内存消耗。后台解析允许 V8 在加载页面时解析脚本。内存可视化工具帮助我们发现，在代码编译后很长一段时间内，后台解析器将使整个区域保持活动状态。通过在编译后立即释放区域，我们显著缩短了区域的生存期，从而降低了平均和峰值内存使用率。

另一个改进是更好地包装田地的结果*抽象语法树*由解析器生成的节点。以前，我们依靠C++编译器在可能的情况下将字段打包在一起。例如，两个布尔值只需要两个位，并且应位于一个单词内或前一个单词的未使用部分内。C++编译器并不总是找到最压缩的打包，因此我们改为手动打包位。这不仅降低了峰值内存使用率，还提高了解析器和编译器的性能。

图5显示了自Chrome 54以来峰值区域内存的改进，与测量的网站相比，平均减少了约40%。

![Figure 5: V8 peak zone memory reduction since Chrome 54 on desktop](../_img/optimizing-v8-memory/peak-zone-memory-reduction.png)

在接下来的几个月里，我们将继续努力减少 V8 的内存占用量。我们为解析器计划了更多的区域内存优化，我们计划专注于512 MB – 1 GB内存范围的设备。

**更新：**上面讨论的所有改进都将Chrome 55的整体内存消耗降低了35%*低内存设备*与Chrome 53相比。其他设备段仅受益于区域内存改进。