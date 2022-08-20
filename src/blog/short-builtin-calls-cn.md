***

标题： “短内置调用”
作者： '[Toon Verwaest](https://twitter.com/tverwaes)，大空头'
化身：

*   toon-verwaest
    日期： 2021-05-06
    标签：
*   JavaScript
    描述： “在 V8 v9.1 中，我们暂时取消了桌面上的内置内存，以避免因远间接调用而导致的性能问题。
    推文：“1394267917013897216”

***

在 V8 v9.1 中，我们暂时禁用[嵌入式内置](https://v8.dev/blog/embedded-builtins)在桌面上。虽然嵌入内置组件可以显著提高内存使用率，但我们已经意识到，嵌入式内置组件和 JIT 编译代码之间的函数调用可能会带来相当大的性能损失。此成本取决于 CPU 的微体系结构。在这篇文章中，我们将解释为什么会发生这种情况，性能是什么样子的，以及我们计划做些什么来长期解决这个问题。

## 代码分配

由 V8 的实时 （JIT） 编译器生成的机器代码在 VM 拥有的内存页上动态分配。V8 在连续的地址空间区域内分配内存页，该区域本身要么随机位于内存中的某个位置（对于[地址空间布局随机化](https://en.wikipedia.org/wiki/Address_space_layout_randomization)原因），或我们分配的 4-GiB 虚拟内存笼内的某个地方[指针压缩](https://v8.dev/blog/pointer-compression).

V8 JIT代码通常调用内置。内置组件本质上是作为 VM 的一部分提供的机器代码片段。有一些内置的内置功能可以实现完整的JavaScript标准库函数，例如[`Function.prototype.bind`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_objects/Function/bind)，但许多内置都是机器代码的帮助器片段，填补了JS的高级语义和CPU的低级功能之间的空白。例如，如果一个 JavaScript 函数想要调用另一个 JavaScript 函数，则该函数实现通常会调用`CallFunction`内置于计算目标JavaScript函数应该如何调用;即，它是代理还是常规函数，它期望多少个参数等。由于这些代码段在我们构建 VM 时是已知的，因此它们被“嵌入”在 Chrome 二进制文件中，这意味着它们最终会进入 Chrome 二进制代码区域。

## 直接呼叫与间接呼叫

在 64 位架构上，包含这些内置组件的 Chrome 二进制文件与 JIT 代码任意远离。随着[x86-64](https://en.wikipedia.org/wiki/X86-64)指令集，这意味着我们不能使用直接调用：它们采用32位签名的立即，用作呼叫地址的偏移量，并且目标可能超过2 GiB。相反，我们需要依靠通过寄存器或内存操作数的间接调用。这种调用更多地依赖于预测，因为从调用指令本身看不出调用的目标是什么。上[ARM64](https://en.wikipedia.org/wiki/AArch64)我们根本不能使用直接调用，因为范围限制为128 MiB。这意味着在这两种情况下，我们都依赖于 CPU 的间接分支预测器的准确性。

## 间接分支预测限制

当以x86-64为目标时，依靠直接呼叫会很好。它应该减少间接分支预测器的压力，因为在指令解码后目标已知，但它也不需要将目标从常量或内存加载到寄存器中。但这不仅仅是机器代码中可见的明显差异。

由于[幽灵 v2](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html)各种设备/操作系统组合已关闭间接分支预测。这意味着在这样的配置上，我们将从依赖于`CallFunction`内置。

更重要的是，即使64位指令集架构（“CPU的高级语言”）支持对远地址的间接调用，微架构也可以自由地实现具有任意限制的优化。间接分支预测器通常假定呼叫距离不超过特定距离（例如，4GiB），每次预测所需的内存较少。例如，[英特尔优化手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-optimization-manual.pdf)明确指出：

> 对于 64 位应用程序，当分支的目标距离分支超过 4 GB 时，分支预测性能可能会受到负面影响。

虽然在 ARM64 上，直接呼叫的体系结构调用范围限制为 128 MiB，但事实证明，[苹果的M1](https://en.wikipedia.org/wiki/Apple_M1)芯片具有相同的微架构4 GiB范围限制，用于间接呼叫预测。对比 4 GiB 更远的呼叫目标的间接呼叫似乎总是被误解。由于特别大[重新订购缓冲区](https://en.wikipedia.org/wiki/Re-order_buffer)的 M1 是 CPU 的组件，它使未来的预测指令能够以推测性无序的方式执行，频繁的误定会导致非常大的性能损失。

## 临时解决方案：复制内置组件

为了避免频繁的错误预测成本，并避免在 x86-64 上尽可能不必要地依赖分支预测，我们决定暂时将内置内容复制到具有足够内存的台式计算机上的 V8 指针压缩笼中。这使复制的内置代码接近动态生成的代码。性能结果在很大程度上取决于设备配置，但以下是我们的性能机器人的一些结果：

![Browsing benchmarks recorded from live pages](/\_img/short-builtin-calls/v8-browsing.svg)

![Benchmark score improvement](/\_img/short-builtin-calls/benchmarks.svg)

取消嵌入内置组件确实会将受影响设备上每个 V8 实例的内存使用量增加 1.2 到 1.4 MiB。作为一个更好的长期解决方案，我们正在研究将JIT代码分配到更接近Chrome二进制文件的位置。通过这种方式，我们可以重新嵌入内置组件以重新获得内存优势，同时进一步提高从V8生成的代码到C++代码的调用的性能。
