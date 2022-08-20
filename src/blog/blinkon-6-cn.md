***

标题：“BlinkOn 6大会上的V8”
作者： 'V8团队'
日期： 2016-07-21 13：33：37
标签：

*   介绍
    描述： “V8 团队在 BlinkOn 6 上的演示概述。

***

BlinkOn是Blink，V8和Chromium贡献者的两年一度的会议。BlinkOn 6于6月16日和6月17日在慕尼黑举行。V8 团队就架构、设计、性能计划和语言实现进行了大量演示。

V8 BlinkOn演讲嵌入在下面。

## 现实世界的 JavaScript 性能

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/xCx4uC7mn6Y" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

*   长度： 31：41
*   [幻灯片](https://docs.google.com/presentation/d/14WZkWbkvtmZDEIBYP5H1GrbC9H-W3nJSg3nvpHwfG5U/edit)

概述了 V8 如何衡量 JavaScript 性能的历史、不同的基准测试时代，以及一种衡量真实世界、流行网站页面负载的新技术，以及每个 V8 组件的详细时间细分。

## 点火：V8的解释器

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/r5OWCtuKiAk" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

*   长度： 36：39
*   [幻灯片](https://docs.google.com/presentation/d/1OqjVqRhtwlKeKfvMdX6HaCIu9wpZsrzqpIVIwQSuiXQ/edit)

介绍 V8 的新 Ignition 解释器，解释整个引擎的架构，以及 Ignition 如何影响内存使用和启动性能。

## 我们如何在 V8 的 GC 中针对 RAIL 进行测量和优化

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/VITAyGT-CJI" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

*   长度： 27：11
*   [幻灯片](https://docs.google.com/presentation/d/15EQ603eZWAnrf4i6QjPP7S3KF3NaL3aAaKhNUEatVzY/edit)

解释 V8 如何使用响应、动画、空闲、加载 （RAIL） 指标来定位低延迟垃圾回收，以及我们最近为减少移动设备上的卡顿而进行的优化。

## ECMAScript 2015 及以后

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/KrGOzEwqRDA" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

*   长度： 28：52
*   [幻灯片](https://docs.google.com/presentation/d/1o1wld5z0BM8RTqXASGYD3Rvov8PzrxySghmrGTYTgw0/edit)

提供有关 V8 中新语言特性的实现、这些特性如何与 Web 平台集成以及继续发展 ECMAScript 语言的标准过程的更新。

## 跟踪从 V8 到 Blink 的包装器（闪电对话）

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/PMDRfYw4UYQ?start=3204" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

*   长度： 2：31
*   [幻灯片](https://docs.google.com/presentation/d/1I6leiRm0ysSTqy7QWh33Gfp7\_y4ngygyM2tDAqdF0fI/edit)

重点介绍 V8 和 Blink 对象之间的跟踪包装器，以及它们如何帮助防止内存泄漏和减少延迟。
