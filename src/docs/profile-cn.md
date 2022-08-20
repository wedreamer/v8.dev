***

## 标题： “使用 V8 基于样本的分析器”&#xA;描述：“本文档介绍如何使用 V8 基于样本的分析器。

V8 具有内置的基于样本的分析功能。默认情况下，性能分析处于关闭状态，但可以通过`--prof`命令行选项。采样器记录 JavaScript 和 C/C++代码的堆栈。

## 建

构建`d8`shell 按照说明进行操作[使用 GN 进行构建](/docs/build-gn).

## 命令行

要开始分析，请使用`--prof`选择。 分析时，V8 会生成一个`v8.log`文件，其中包含分析数据。

窗户：

```bash
build\Release\d8 --prof script.js
```

其他平台（替换`ia32`跟`x64`如果要分析`x64`构建）：

```bash
out/ia32.release/d8 --prof script.js
```

## 处理生成的输出

日志文件处理是使用由 d8 shell 运行的 JS 脚本完成的。为此，一个`d8`二进制（或符号链接，或`d8.exe`在 Windows 上），必须位于 V8 检出的根目录中，或位于环境变量指定的路径中`D8_PATH`.注意：此二进制文件仅用于处理日志，但不用于实际分析，因此它是什么版本等并不重要。

**确保`d8`用于分析不是用`is_component_build`!**

窗户：

```bash
tools\windows-tick-processor.bat v8.log
```

Linux：

```bash
tools/linux-tick-processor v8.log
```

macOS：

```bash
tools/mac-tick-processor v8.log
```

## 网页 UI 用于`--prof`

预处理日志`--preprocess`（解决C++符号等）。

```bash
$V8_PATH/tools/linux-tick-processor --preprocess > v8.json
```

打开[`tools/profview/index.html`](https://v8.dev/tools/head/profview)，然后选择`v8.json`文件在那里。

## 示例输出

    Statistical profiling result from benchmarks\v8.log, (4192 ticks, 0 unaccounted, 0 excluded).

     [Shared libraries]:
       ticks  total  nonlib   name
          9    0.2%    0.0%  C:\WINDOWS\system32\ntdll.dll
          2    0.0%    0.0%  C:\WINDOWS\system32\kernel32.dll

     [JavaScript]:
       ticks  total  nonlib   name
        741   17.7%   17.7%  LazyCompile: am3 crypto.js:108
        113    2.7%    2.7%  LazyCompile: Scheduler.schedule richards.js:188
        103    2.5%    2.5%  LazyCompile: rewrite_nboyer earley-boyer.js:3604
        103    2.5%    2.5%  LazyCompile: TaskControlBlock.run richards.js:324
         96    2.3%    2.3%  Builtin: JSConstructCall
        ...

     [C++]:
       ticks  total  nonlib   name
         94    2.2%    2.2%  v8::internal::ScavengeVisitor::VisitPointers
         33    0.8%    0.8%  v8::internal::SweepSpace
         32    0.8%    0.8%  v8::internal::Heap::MigrateObject
         30    0.7%    0.7%  v8::internal::Heap::AllocateArgumentsObject
        ...


     [GC]:
       ticks  total  nonlib   name
        458   10.9%

     [Bottom up (heavy) profile]:
      Note: percentage shows a share of a particular caller in the total
      amount of its parent calls.
      Callers occupying less than 2.0% are not shown.

       ticks parent  name
        741   17.7%  LazyCompile: am3 crypto.js:108
        449   60.6%    LazyCompile: montReduce crypto.js:583
        393   87.5%      LazyCompile: montSqrTo crypto.js:603
        212   53.9%        LazyCompile: bnpExp crypto.js:621
        212  100.0%          LazyCompile: bnModPowInt crypto.js:634
        212  100.0%            LazyCompile: RSADoPublic crypto.js:1521
        181   46.1%        LazyCompile: bnModPow crypto.js:1098
        181  100.0%          LazyCompile: RSADoPrivate crypto.js:1628
        ...

## 分析 Web 应用程序

当今高度优化的虚拟机可以以极快的速度运行 Web 应用。但是，人们不应该仅仅依靠它们来实现出色的性能：经过精心优化的算法或更便宜的功能通常可以在所有浏览器上实现数倍的速度提升。[Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/)'[CPU 探查器](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/reference)帮助您分析代码的瓶颈。但有时，您需要更深入、更精细地了解：这就是 V8 的内部分析器派上用场的地方。

让我们使用该探查器来检查[曼德布洛特探险家演示](https://web.archive.org/web/20130313064141/http://ie.microsoft.com/testdrive/performance/mandelbrotexplorer/)那 微软[释放](https://blogs.msdn.microsoft.com/ie/2012/11/13/ie10-fast-fluid-perfect-for-touch-and-available-now-for-windows-7/)与 IE10 一起。在演示版发布后，V8修复了一个不必要地减慢计算速度的错误（因此在演示的博客文章中Chrome的性能很差），并进一步优化了引擎，实现了更快的速度`exp()`比标准系统库提供的近似值。根据这些更改，**该演示的运行速度比以前测量的快 8×**在浏览器中。

但是，如果您希望代码在所有浏览器上更快地运行，该怎么办？你应该首先**了解是什么让您的 CPU 忙碌**.运行 Chrome（Windows 和 Linux）[金丝雀](https://tools.google.com/dlpage/chromesxs)）， 并使用以下命令行开关，这会导致它输出探查器刻度信息（在`v8.log`file）用于您指定的URL，在我们的例子中，这是没有Web工作者的Mandelbrot演示的本地版本：

```bash
./chrome --js-flags='--prof' --no-sandbox 'http://localhost:8080/'
```

准备测试用例时，请确保它在加载时立即开始工作，并在计算完成后关闭Chrome（按Alt + F4），以便您在日志文件中只有您关心的刻度。另请注意，尚未使用此技术正确分析 Web 工作线程。

然后，处理`v8.log`文件与`tick-processor`V8（或新的实用 Web 版本）附带的脚本：

```bash
v8/tools/linux-tick-processor v8.log
```

以下是已处理输出的一个有趣的片段，应该会引起您的注意：

    Statistical profiling result from null, (14306 ticks, 0 unaccounted, 0 excluded).
     [Shared libraries]:
       ticks  total  nonlib   name
       6326   44.2%    0.0%  /lib/x86_64-linux-gnu/libm-2.15.so
       3258   22.8%    0.0%  /.../chrome/src/out/Release/lib/libv8.so
       1411    9.9%    0.0%  /lib/x86_64-linux-gnu/libpthread-2.15.so
         27    0.2%    0.0%  /.../chrome/src/out/Release/lib/libwebkit.so

顶部显示 V8 在特定于操作系统的系统库内花费的时间比在自己的代码中花费的时间更多。让我们通过检查“自下而上”的输出部分来看看它的原因，您可以在其中将缩进的行读作“被调用者”（以及以`*`表示该功能已通过 TurboFan 进行了优化）：

    [Bottom up (heavy) profile]:
      Note: percentage shows a share of a particular caller in the total
      amount of its parent calls.
      Callers occupying less than 2.0% are not shown.

       ticks parent  name
       6326   44.2%  /lib/x86_64-linux-gnu/libm-2.15.so
       6325  100.0%    LazyCompile: *exp native math.js:91
       6314   99.8%      LazyCompile: *calculateMandelbrot http://localhost:8080/Demo.js:215

超过**总时间的 44% 用于执行`exp()`系统库中的功能**!为调用系统库增加了一些开销，这意味着大约三分之二的总时间用于评估`Math.exp()`.

如果你看一下JavaScript代码，你会发现`exp()`仅用于生成平滑的灰度调色板。有无数种方法可以产生平滑的灰度调色板，但让我们假设你真的非常喜欢指数梯度。这就是算法优化发挥作用的地方。

您会注意到`exp()`使用区域中的参数调用`-4 < x < 0`，因此我们可以安全地将其替换为[泰勒近似](https://en.wikipedia.org/wiki/Taylor_series)对于该范围，它仅通过乘法和几个除法提供相同的平滑梯度：

    exp(x) ≈ 1 / ( 1 - x + x * x / 2) for -4 < x < 0

与最新的金丝雀相比，以这种方式调整算法可将性能提高30%，与基于系统库相比，性能可提高5×`Math.exp()`在铬金丝雀上。

![](/\_img/docs/profile/mandelbrot.png)

此示例显示了 V8 的内部分析器如何帮助您更深入地了解代码瓶颈，以及更智能的算法可以进一步提高性能。

要了解有关如何对当今复杂且要求苛刻的 Web 应用程序进行基准测试的更多信息，请阅读[V8 如何衡量实际性能](/blog/real-world-performance).
