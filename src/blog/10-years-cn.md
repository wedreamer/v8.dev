***

标题：“庆祝V8成立10周年”
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)），V8历史学家
化身：

*   'mathias-bynens'
    发布日期： 2018-09-11 19：00：00
    标签：
*   基准
    描述： “过去10年以及之前的几年中V8项目的主要里程碑概述，当时该项目仍然是秘密的。
    推文：“1039559389324238850”

***

本月标志着不仅谷歌Chrome，而且V8项目发布10周年。这篇文章概述了过去10年以及之前的几年中V8项目的主要里程碑，当时该项目仍然是秘密的。

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/G0vnrPTuxZA" width="640" height="360" loading="lazy"></iframe>
  </div>
  <figcaption>A visualization of the V8 code base over time, created using <a href="http://gource.io/"><code>gource</code></a>.</figcaption>
</figure>

## V8出货前：早年

谷歌雇用[拉尔斯·巴克](https://en.wikipedia.org/wiki/Lars_Bak\_%28computer_programmer%29)在秋天**2006**为Chrome Web浏览器构建一个新的JavaScript引擎，这在当时仍然是一个秘密的内部Google项目。拉尔斯最近从硅谷搬回了丹麦的奥胡斯。由于那里没有谷歌办公室，而拉尔斯想留在丹麦，拉尔斯和该项目的几位原始工程师开始在他农场的附属建筑中从事该项目。新的JavaScript运行时被命名为“V8”，这是对经典肌肉车中强大引擎的有趣参考。后来，当 V8 团队成长起来时，开发人员从他们简陋的地方搬到了奥胡斯的一座现代化的办公楼，但团队带着他们独特的动力，专注于构建这个星球上最快的 JavaScript 运行时。

## 推出并演进 V8

V8 在同一天开源[Chrome 推出](https://blog.chromium.org/2008/09/welcome-to-chromium\_02.html)：9月2日，**2008**.[初始提交](https://chromium.googlesource.com/v8/v8/+/43d26ecc3563a46f62a0224030667c8f8f3f6ceb)可追溯到2008年6月30日。在此之前，V8 开发是在私有 CVS 存储库中进行的。最初，V8 仅支持 ia32 和 ARM 指令集，并使用[断续器](https://scons.org/)作为其构建系统。

**2009**见证了一个全新的正则表达式引擎的引入，命名为[Irregexp](https://blog.chromium.org/2009/02/irregexp-google-chromes-new-regexp.html)，从而改进了实际正则表达式的性能。随着 x64 端口的引入，支持的指令集数量从两个增加到三个。2009年也标志着[Node.js项目的第一个版本](https://github.com/nodejs/node-v0.x-archive/releases/tag/v0.0.1)，它嵌入了 V8。非浏览器项目嵌入 V8 的可能性是[明确提及](https://www.google.com/googlebooks/chrome/big\_16.html)在原始的Chrome漫画中。有了Node.js，它真的发生了！Node.js成长为最受欢迎的JavaScript生态系统之一。

**2010**随着 V8 引入了全新的优化 JIT 编译器，运行时性能有了很大的提升。[曲轴](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html)生成的机器代码比以前（未命名的）V8 编译器快两倍，小 30%。同年，V8增加了第四个指令集：32位MIPS。

**2011**来了，垃圾收集得到了极大的改善。[新的增量垃圾回收器](https://blog.chromium.org/2011/11/game-changer-for-interactive.html)大幅缩短了暂停时间，同时保持了出色的峰值性能和低内存使用率。V8 引入了 Isolates 的概念，它允许嵌入者在一个进程中启动 V8 运行时的多个实例，为 Chrome 中轻量级的 Web Worker 铺平了道路。V8 的两个构建系统迁移中的第一个发生在我们从 SCons 过渡到[吉普](https://gyp.gsrc.io/).我们实现了对 ES5 严格模式的支持。与此同时，在新的领导下，开发从奥胡斯转移到慕尼黑（德国），奥胡斯的原始团队进行了大量异花授粉。

**2012**是V8项目基准测试的一年。该团队进行了速度冲刺，以优化V8的性能，通过[太阳蜘蛛](https://webkit.org/perf/sunspider/sunspider.html)和[海肯](https://krakenbenchmark.mozilla.org/)基准测试套件。后来，我们开发了一个新的基准套件，名为[辛烷](https://chromium.github.io/octane/)（与[V8 长凳](http://www.netchain.com/Tools/v8/)在其核心），将峰值性能竞争带到了最前沿，并刺激了所有主要JS引擎中运行时和JIT技术的大规模改进。这些努力的一个成果是从随机抽样切换到确定性的、基于计数的技术，用于检测 V8 运行时分析器中的“热”函数。这使得某些页面加载（或基准测试运行）随机比其他页面加载（或基准测试运行）慢得多的可能性大大降低。

**2013**见证了 JavaScript 的低级子集的出现，命名为[asm.js](http://asmjs.org/).由于 asm.js 仅限于静态类型的算术、函数调用和仅使用基元类型的堆访问，因此经过验证的 asm.js代码可以以可预测的性能运行。我们发布了新版本的辛烷，[辛烷值 2.0](https://blog.chromium.org/2013/11/announcing-octane-20.html)更新现有基准测试以及针对 asm.js等用例的新基准测试。Octane 推动了新编译器优化的发展，例如[分配折叠](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/42478.pdf)和[基于分配站点的类型转换和预增强优化](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43823.pdf)大大提高了峰值性能。作为我们内部昵称为“Handlepocalypse”的努力的一部分，V8 Handle API被完全重写，使其更易于正确安全地使用。同样在2013年，Chrome实施了`TypedArray`s 在 JavaScript 中是[从眨眼移动到 V8](https://codereview.chromium.org/13064003).

在**2014**，V8 将 JIT 编译的一些工作从主线程上移走了[并发编译](https://blog.chromium.org/2014/02/compiling-in-background-for-smoother.html)，减少卡顿并显著提高性能。那年晚些时候，我们[降落](https://github.com/v8/v8/commit/a1383e2250dc5b56b777f2057f1600537f02023e)名为TurboFan的新优化编译器的初始版本。同时，我们的合作伙伴帮助将 V8 移植到三个新的指令集架构：PPC、MIPS64 和 ARM64。在Chromium之后，V8过渡到另一个构建系统，[断续器](https://gn.googlesource.com/gn/#gn).V8 测试基础设施取得了重大改进，并有*Tryserver*现在可用于在登陆之前在各种构建机器人上测试每个补丁。对于源代码管理，V8 从 SVN 迁移到 Git。

**2015**对于V8来说，这是繁忙的一年。我们实施了[代码缓存和脚本流](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html)，显著加快了网页加载时间。我们的运行时系统使用分配纪念品的工作是[发表于 ISMM 2015](https://ai.google/research/pubs/pub43823).那年晚些时候，我们[拉开 序幕](https://github.com/v8/v8/commit/7877c4e0c77b5c2b97678406eab7e9ad6eba4a4d)在一个名为Ignition的新解释器上的工作。我们尝试了将 JavaScript 子集化的想法[强模式](https://docs.google.com/document/d/1Qk0qC4s_XNCLemj42FqfsRLp49nDQMZ1y7fwf5YjaI4/view)以实现更强大的保证和更可预测的性能。我们在标志后面实施了强模式，但后来发现它的好处并不能证明成本是合理的。添加[提交队列](https://dev.chromium.org/developers/testing/commit-queue)在生产力和稳定性方面有了很大的提高。V8的垃圾回收器也开始与Blink等嵌入器合作，在空闲期间安排垃圾回收工作。[空闲时间垃圾回收](/blog/free-garbage-collection)显著减少了可观察的垃圾回收卡顿和内存消耗。12月，[第一个WebAssembly原型](https://github.com/titzer/v8-native-prototype)登陆V8。

在**2016**，我们发布了ES2015（以前称为“ES6”）功能集的最后部分（包括承诺，类语法，词法范围，解构等），以及一些ES2016功能。我们还开始推出新的Ignition和TurboFan管道，用它来[编译和优化 ES2015 和 ES2016 功能](/blog/v8-release-56)，并默认为[低端安卓设备](/blog/ignition-interpreter).我们在空闲时间垃圾回收方面的成功工作是在[2016年波兰迪展](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45361.pdf).我们开始了[奥里诺科项目](/blog/orinoco)，一个新的用于 V8 的并行和并发垃圾回收器，以减少主线程垃圾回收时间。在一次重大的重新聚焦中，我们将性能工作从合成微观基准转移到了合成的微观基准测试上，而是开始认真地衡量和优化。[实际性能](/blog/real-world-performance).为了进行调试，V8 检查器是[迁移](/blog/v8-release-55)从Chromium到V8，允许任何V8嵌入器（而不仅仅是Chromium）使用Chrome DevTools来调试在V8中运行的JavaScript。WebAssembly原型与其他浏览器供应商协调，从原型发展到实验支持[对WebAssembly的实验性支持](/blog/webassembly-experimental).已接收 V8[ACM SIGPLAN 编程语言软件奖](http://www.sigplan.org/Awards/Software/).并添加了另一个端口：S390。

在**2017**，我们终于完成了对发动机的多年大修，使新的[点火和涡轮风扇](/blog/launching-ignition-and-turbofan)默认情况下，管道。这使得以后可以拆卸曲轴（[130，380 行已删除的代码](https://chromium-review.googlesource.com/c/v8/v8/+/547717)） 和[全码](https://chromium-review.googlesource.com/c/v8/v8/+/584773)从代码库。我们推出了Orinoco v1.0，包括[并发标记](/blog/concurrent-marking)、并发扫描、并行清理和并行压缩。我们正式认可Node.js与Chromium一起作为一流的V8嵌入器。从那时起，如果V8补丁破坏了Node.js测试套件，那么它就不可能登陆。我们的基础架构获得了对正确性模糊测试的支持，确保任何一段代码都能产生一致的结果，无论其运行在何种配置中。

在全行业范围内的协调发布中，V8[默认发货 WebAssembly](/blog/v8-release-57).我们实施了对[JavaScript 模块](/features/modules)以及完整的 ES2017 和 ES2018 功能集（包括异步函数、共享内存、异步迭代、休息/传播属性和 RegExp 功能）。我们发货[对 JavaScript 代码覆盖率的本机支持](/blog/javascript-code-coverage)，并启动了[网络工具基准测试](/blog/web-tooling-benchmark)帮助我们衡量 V8 的优化如何影响实际开发人员工具的性能以及它们生成的 JavaScript 输出。[包装器跟踪](/blog/tracing-js-dom)从JavaScript对象到C++DOM对象，然后返回，使我们能够解决Chrome中长期存在的内存泄漏，并有效地处理JavaScript和Blink堆上对象的传递闭包。我们稍后使用此基础结构来增强堆快照开发人员工具的功能。

**2018**看到一个全行业的安全事件颠覆了我们认为我们对CPU信息安全的了解，公开披露了[幽灵/崩溃漏洞](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html).V8 工程师执行了广泛的攻击性研究，以帮助了解托管语言的威胁并开发缓解措施。V8 出货[缓解措施](/docs/untrusted-code-mitigations)针对运行不可信代码的嵌入器的 Spectre 和类似的侧信道攻击。

最近，我们发布了一个名为 WebAssembly 的基线编译器。[起飞](/blog/liftoff)这大大减少了WebAssembly应用程序的启动时间，同时仍然实现了可预测的性能。我们发货[`BigInt`](/blog/bigint)，一个新的 JavaScript 原语，它支持[任意精度整数](/features/bigint).我们实施了[嵌入式内置](/blog/embedded-builtins)，并使其成为可能[懒惰地反序列化它们](/blog/lazy-deserialization)，显著减少了 V8 在多个隔离器上的占用空间。我们使[在后台线程上编译脚本字节码](/blog/background-compilation).我们开始了[统一 V8-Blink 堆项目](https://docs.google.com/presentation/d/12ZkJ0BZ35fKXtpM342PmKM5ZSxPt03\_wsRgbsJYl3Pc)以同步运行跨组件 V8 和 Blink 垃圾回收。这一年还没有结束...

## 性能起伏

Chrome 多年来的 V8 Bench 得分显示了 V8 更改对性能的影响。（我们使用的是 V8 Bench，因为它是为数不多的仍然可以在原始 Chrome 测试版中运行的基准测试之一。

![Chrome’s V8 Bench score from 2008 to 2018](../_img/10-years/v8-bench.svg)

我们在这个基准上的分数上升了**4×**在过去的十年里！

但是，您可能会注意到多年来有两次性能下降。两者都很有趣，因为它们对应于V8历史上的重大事件。2015 年的性能下降发生在 V8 发布 ES2015 功能的基准版本时。这些特性在 V8 代码库中是跨领域的，因此我们专注于其初始版本的正确性而不是性能。我们接受了这些轻微的速度回归，以便尽快将功能提供给开发人员。2018年初，Spectre漏洞被披露，V8发布了缓解措施来保护用户免受潜在攻击，导致性能再次下降。幸运的是，现在Chrome正在发货[站点隔离](https://developers.google.com/web/updates/2018/07/site-isolation)，我们可以再次禁用缓解措施，使性能恢复正常。

从这张图表中得出的另一个结论是，它在2013年左右开始趋于平稳。这是否意味着V8放弃了并停止了对性能的投资？恰恰相反！图形的扁平化代表了 V8 团队从合成微基准测试（如 V8 Bench 和 Octane）到优化[实际性能](/blog/real-world-performance).V8 Bench是一个旧的基准测试，它不使用任何现代JavaScript功能，也不接近实际的现实世界生产代码。将此与较新的速度计基准测试套件进行对比：

![Chrome’s Speedometer 1 score from 2013 to 2018](../_img/10-years/speedometer-1.svg)

尽管V8 Bench从2013年到2018年几乎没有改进，但我们的车速表1得分却上升了（另一个）**4×**在同一时间段内。（我们使用 Speedometer 1 是因为 Speedometer 2 使用了 2013 年尚不支持的现代 JavaScript 功能。

如今，我们有[甚至更好](/blog/speedometer-2) [基准](/blog/web-tooling-benchmark)更准确地反映了现代JavaScript应用程序，最重要的是，我们[主动测量和优化现有 Web 应用](https://www.youtube.com/watch?v=xCx4uC7mn6Y).

## 总结

虽然V8最初是为Google Chrome构建的，但它一直是一个独立的项目，具有单独的代码库和嵌入API，允许任何程序使用其JavaScript执行服务。在过去的10年中，该项目的开放性帮助它不仅成为Web平台的关键技术，而且在Node.js等其他上下文中也是如此。在此过程中，尽管发生了许多变化和戏剧性的增长，但该项目不断发展并保持相关性。

最初，V8 仅支持两个指令集。在过去的10年中，支持的平台列表达到了八个：ia32，x64，ARM，ARM64，32位和64位MIPS，64位PPC和S390。V8 的构建系统从 SCons 迁移到 GYP 再到 GN。该项目从丹麦转移到德国，现在工程师遍布世界各地，包括伦敦，山景城和旧金山，谷歌以外的贡献者来自更多地方。我们已经将整个JavaScript编译管道从未命名的组件转换为Full-codegen（基线编译器）和Crankshaft（反馈驱动的优化编译器）到Ignition（解释器）和TurboFan（更好的反馈驱动的优化编译器）。V8从“只是”一个JavaScript引擎变成了支持WebAssembly。JavaScript语言本身从ECMAScript 3发展到ES2018;最新的V8甚至实现了ES2018之后的功能。

Web的故事弧线是一个漫长而持久的故事。庆祝Chrome和V8的10岁生日是一个很好的机会，可以反映出尽管这是一个重要的里程碑，但Web平台的叙述已经持续了超过25年。我们毫不怀疑，网络的故事至少将在未来持续那么长时间。我们致力于确保 V8、JavaScript 和 WebAssembly 继续成为这种叙事中有趣的角色。我们很高兴看到未来十年会发生什么。敬请期待！
