***

标题：“Chrome的一小步，V8的一大堆”
作者：“堆的守护者乌兰·德根巴耶夫，汉内斯·佩耶，迈克尔·利普茨和DevTools战士阿列克谢·科齐亚廷斯基”
化身：

*   '乌兰-德根巴耶夫'
*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2017-02-09 13：33：37
    标签：
*   记忆
    描述：“V8 最近增加了对堆大小的硬限制。

***

V8 对其堆大小有硬性限制。这可以防止应用程序出现内存泄漏。当应用程序达到此硬限制时，V8 会执行一系列最后手段的垃圾回收。如果垃圾回收无助于释放内存，V8 将停止执行并报告内存不足故障。如果没有硬限制，内存泄漏的应用程序可能会耗尽所有系统内存，从而损害其他应用程序的性能。

具有讽刺意味的是，这种保护机制使JavaScript开发人员更难调查内存泄漏。在开发人员设法检查 DevTools 中的堆之前，应用程序可能会耗尽内存。此外，DevTools进程本身可能会耗尽内存，因为它使用普通的V8实例。例如，拍摄堆快照[此演示](https://ulan.github.io/misc/heap-snapshot-demo.html)由于当前稳定的 Chrome 上的内存不足而中止执行。

从历史上看，V8 堆限制被方便地设置为适合有符号的 32 位整数范围，并带有一些边距。随着时间的推移，这种便利性导致V8中的代码草率，混合了不同位宽度的类型，有效地打破了增加限制的能力。最近，我们清理了垃圾回收器代码，允许使用更大的堆大小。DevTools已经利用了此功能，并在前面提到的演示中拍摄堆快照，在最新的Chrome Canary中按预期工作。

我们还在 DevTools 中添加了一项功能，以便在应用程序接近内存不足时暂停应用程序。此功能可用于调查导致应用程序在短时间内分配大量内存的 Bug。运行时[此演示](https://ulan.github.io/misc/oom.html)使用最新的Chrome Canary，DevTools在内存不足故障之前暂停应用程序并增加堆限制，使用户有机会检查堆，在控制台上评估表达式以释放内存，然后恢复执行以进行进一步调试。

![](../_img/heap-size-limit/debugger.png)

V8 嵌入器可以使用[`set_max_old_space_size`](https://codesearch.chromium.org/chromium/src/v8/include/v8.h?q=set_max_old_space_size)的功能`ResourceConstraints`应用程序接口。但请注意，垃圾回收器中的某些阶段对堆大小具有线性依赖性。垃圾回收暂停可能会随着堆的增大而增加。