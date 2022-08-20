***

标题： 'Jank Busters Part One'
作者：“Jank破坏者：Jochen Eisinger，Michael Lippautz和Hannes Payer”
化身：

*   “迈克尔-利普茨”
*   “hannes-payer”
    日期： 2015-10-30 13：33：37
    标签：
*   记忆
    描述： “本文讨论了在Chrome 41和Chrome 46之间实施的优化，这些优化显着减少了垃圾回收暂停，从而改善了用户体验。

***

当Chrome无法在16.66毫秒内渲染帧（中断每秒60帧的运动）时，可以注意到Jank，换句话说，可见的口吃。截至今天，大多数V8垃圾回收工作都是在主渲染线程c.f.上执行的。图 1，当需要维护的对象太多时，通常会导致卡顿。消除卡顿一直是 V8 团队的首要任务（[1](https://blog.chromium.org/2011/11/game-changer-for-interactive.html),[2](https://www.youtube.com/watch?v=3vPOlGRH6zk),[3](/blog/free-garbage-collection)).本文讨论了在Chrome 41和Chrome 46之间实现的一些优化，这些优化可显着减少垃圾回收暂停，从而获得更好的用户体验。

![Figure 1: Garbage collection performed on the main thread](/\_img/jank-busters/gc-main-thread.png)

垃圾回收期间卡顿的一个主要来源是处理各种簿记数据结构。其中许多数据结构支持与垃圾回收无关的优化。两个示例是所有 ArrayBuffer 的列表，以及每个 ArrayBuffer 的视图列表。这些列表允许有效地实现 DetachArrayBuffer 操作，而不会对访问 ArrayBuffer 视图造成任何性能影响。但是，在网页创建数百万个 ArrayBuffers（例如，基于 WebGL 的游戏）的情况下，在垃圾回收期间更新这些列表会导致严重的卡顿。在Chrome 46中，我们删除了这些列表，而是通过在每次加载和存储到ArrayBuffers之前插入检查来检测分离的缓冲区。这通过在整个程序执行过程中分散来摊销在GC期间完成大簿记列表的成本，从而减少卡顿。虽然每次访问检查理论上会减慢大量使用ArrayBuffers的程序的吞吐量，但实际上V8的优化编译器通常可以删除冗余检查并将剩余的检查提升到循环之外，从而产生更流畅的执行配置文件，几乎没有或没有整体性能损失。

卡顿的另一个来源是与跟踪Chrome和V8之间共享对象的生命周期相关的簿记。尽管Chrome和V8内存堆是不同的，但它们必须针对某些对象（如DOM节点）进行同步，这些对象在Chrome的C++代码中实现，但可以从JavaScript访问。V8 创建了一个称为句柄的不透明数据类型，该数据类型允许 Chrome 在不知道实现的任何细节的情况下操作 V8 堆对象。对象的生存期与句柄绑定：只要 Chrome 保留句柄，V8 的垃圾回收器就不会丢弃该对象。V8 为每个通过 V8 API 传回 Chrome 的句柄创建一个称为全局引用的内部数据结构，这些全局引用告诉 V8 的垃圾回收器该对象仍然处于活动状态。对于WebGL游戏，Chrome可能会创建数百万个这样的句柄，而V8反过来需要创建相应的全局引用来管理它们的生命周期。在主垃圾回收暂停中处理这些大量全局引用是可观察到的卡顿。幸运的是，传递给WebGL的对象通常只是传递而从未实际修改过，从而实现了简单的静态。[逃逸分析](https://en.wikipedia.org/wiki/Escape_analysis).从本质上讲，对于已知通常将小数组作为参数的WebGL函数，基础数据被复制到堆栈上，从而使全局引用过时。这种混合方法的结果是，对于渲染繁重的WebGL游戏，暂停时间最多减少了50%。

V8 的大部分垃圾回收都是在主渲染线程上执行的。将垃圾回收操作移动到并发线程可减少垃圾回收器的等待时间，并进一步减少卡顿。这是一个本质上复杂的任务，因为主JavaScript应用程序和垃圾回收器可以同时观察和修改相同的对象。到目前为止，并发仅限于扫描旧一代的常规对象 JS 堆。最近，我们还实现了对 V8 堆的代码和映射空间的并发扫描。此外，我们还实现了未使用页面的并发取消映射，以减少必须在主线程上执行的工作，c.f. 图 2。

![Figure 2: Some garbage collection operations performed on the concurrent garbage collection threads.](/\_img/jank-busters/gc-concurrent-threads.png)

所讨论的优化的影响在基于WebGL的游戏中清晰可见，例如[Turbolenz的Oort Online演示](http://oortonline.gl/).以下视频将 Chrome 41 与 Chrome 46 进行了比较：

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/PgrCJpbTs9I" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

我们目前正在使更多的垃圾回收组件成为增量，并发和并行，以进一步缩短主线程上的垃圾回收暂停时间。请继续关注，因为我们有一些有趣的补丁正在筹备中。
