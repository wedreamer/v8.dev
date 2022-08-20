***

标题： “V8 中的指针压缩”
作者：伊戈尔·谢卢德科和圣地亚哥·阿博伊·索拉内斯，*这*指针式压缩机
化身：

*   “伊戈尔-谢卢德科”
*   “圣地亚哥-阿博伊-索兰内斯”
    日期： 2020-03-30
    标签：
*   内部
*   记忆
    描述： 'V8 将其堆大小减小了 43%！在“V8 中的指针压缩”中了解如何操作！
    推文：“1244653541379182596”

***

记忆力和性能之间不断发生着一场战斗。作为用户，我们希望事情变得快速，并消耗尽可能少的内存。不幸的是，通常提高性能是以内存消耗为代价的（反之亦然）。

早在2014年，Chrome就从32位进程切换到64位进程。这给了Chrome更好的[安全性、稳定性和性能](https://blog.chromium.org/2014/08/64-bits-of-awesome-64-bit-windows\_26.html)，但它需要内存成本，因为每个指针现在占用八个字节而不是四个字节。我们接受了在 V8 中减少此开销的挑战，以尝试恢复尽可能多的浪费的 4 个字节。

在深入实施之前，我们需要知道我们的立场，以正确评估情况。为了测量我们的记忆力和表现，我们使用一组[网页](https://v8.dev/blog/optimizing-v8-memory)反映了流行的现实世界网站。数据显示，V8贡献了Chrome的60%。[渲染器进程](https://www.chromium.org/developers/design-documents/multi-process-architecture)台式机上的内存消耗，平均为 40%。

![V8 memory consumption percentage in Chrome’s renderer memory](/\_img/pointer-compression/memory-chrome.svg)

指针压缩是 V8 中为减少内存消耗而正在进行的几项工作之一。这个想法非常简单：我们可以存储来自某个“基”地址的32位偏移，而不是存储64位指针。有了这样一个简单的想法，我们可以从V8的这种压缩中获得多少收益？

V8 堆包含大量项目，例如浮点值、字符串字符、解释器字节码和标记值（有关详细信息，请参阅下一节）。在检查堆时，我们发现在现实世界的网站上，这些标记的值占据了V8堆的70%左右！

让我们仔细看看什么是标记值。

## V8 中的值标记

V8 中的 JavaScript 值表示为对象并在 V8 堆上分配，无论它们是对象、数组、数字还是字符串。这允许我们将任何值表示为指向对象的指针。

许多 JavaScript 程序对整数值执行计算，例如在循环中递增索引。为了避免我们每次整数递增时都必须分配一个新的数字对象，V8 使用众所周知的[指针标记](https://en.wikipedia.org/wiki/Tagged_pointer)用于在 V8 堆指针中存储其他或备用数据的技术。

标记位具有双重用途：它们向位于 V8 堆中的对象发出强/弱指针或小整数的信号。因此，整数的值可以直接存储在标记的值中，而不必为其分配额外的存储空间。

V8 始终在堆中以字对齐的地址分配对象，这允许它使用 2 个（或 3 个，取决于机器字大小）最低有效位进行标记。在 32 位体系结构上，V8 使用最低有效位将 Smis 与堆对象指针区分开来。对于堆指针，它使用第二个最低有效位来区分强引用和弱引用：

<pre>
                        |----- 32 bits -----|
Pointer:                |_____address_____<b>w1</b>|
Smi:                    |___int31_value____<b>0</b>|
</pre>

哪里*w*有点用于区分强指针和弱指针。

请注意，Smi 值只能携带 31 位有效负载，包括符号位。对于指针，我们有 30 位可以用作堆对象地址有效负载。由于字对齐，分配粒度为 4 个字节，这为我们提供了 4 GB 的可寻址空间。

在 64 位架构上，V8 值如下所示：

<pre>
            |----- 32 bits -----|----- 32 bits -----|
Pointer:    |________________address______________<b>w1</b>|
Smi:        |____int32_value____|000000000000000000<b>0</b>|
</pre>

您可能会注意到，与 32 位架构不同，在 64 位架构上，V8 可以使用 32 位作为 Smi 值有效负载。以下各节将讨论 32 位 Smis 对指针压缩的影响。

## 压缩的标记值和新的堆布局

使用指针压缩，我们的目标是以某种方式将两种标记值放入 64 位体系结构上的 32 位中。我们可以通过以下方式将指针放入32位：

*   确保所有 V8 对象都分配在 4 GB 内存范围内
*   将指针表示为此范围内的偏移量

有这样的硬限制是不幸的，但是Chrome中的V8已经对V8堆的大小有2 GB或4 GB的限制（取决于底层设备的强大程度），即使在64位架构上也是如此。其他 V8 嵌入器（如 Node.js）可能需要更大的堆。如果我们施加最大 4 GB，则意味着这些嵌入器不能使用指针压缩。

现在的问题是如何更新堆布局以确保 32 位指针唯一标识 V8 对象。

### 琐碎的堆布局

简单的压缩方案是在前 4 GB 的地址空间中分配对象。

![Trivial heap layout](/\_img/pointer-compression/heap-layout-0.svg)

不幸的是，这不是V8的选项，因为Chrome的渲染器进程可能需要在同一渲染器进程中创建多个V8实例，例如对于Web/Service Workers。否则，使用此方案，所有这些 V8 实例都会争用相同的 4 GB 地址空间，因此所有 V8 实例一起施加了 4 GB 的内存限制。

### 堆布局，v1

如果我们将 V8 的堆排列在其他地方的连续 4 GB 地址空间区域中，则**符号**从基极的 32 位偏移量唯一标识指针。

<figure>
  <img src="/_img/pointer-compression/heap-layout-1.svg" width="827" height="323" alt="" loading="lazy">
  <figcaption>Heap layout, base aligned to start</figcaption>
</figure>

如果我们还要确保基数为 4 GB 对齐，则所有指针的上限 32 位都相同：

                |----- 32 bits -----|----- 32 bits -----|
    Pointer:    |________base_______|______offset_____w1|

我们还可以通过将Smi有效载荷限制为31位并将其放置在较低的32位来使Smis可压缩。基本上，使它们类似于32位架构上的Smis。

             |----- 32 bits -----|----- 32 bits -----|
    Smi:     |sssssssssssssssssss|____int31_value___0|

哪里*s*是 Smi 有效负载的符号值。如果我们有一个符号扩展表示，我们只需要64位单词的一点算术平移来压缩和解压缩Smis。

现在，我们可以看到指针和 Smis 的上半个字完全由下半个字定义。然后，我们可以将后者仅存储在内存中，将存储标记值所需的内存减少一半：

                        |----- 32 bits -----|----- 32 bits -----|
    Compressed pointer:                     |______offset_____w1|
    Compressed Smi:                         |____int31_value___0|

假设基数为 4 GB 对齐，则压缩只是截断：

```cpp
uint64_t uncompressed_tagged;
uint32_t compressed_tagged = uint32_t(uncompressed_tagged);
```

但是，解压缩代码稍微复杂一些。我们需要区分符号扩展Smi和零扩展指针，以及是否在基数中添加。

```cpp
uint32_t compressed_tagged;

uint64_t uncompressed_tagged;
if (compressed_tagged & 1) {
  // pointer case
  uncompressed_tagged = base + uint64_t(compressed_tagged);
} else {
  // Smi case
  uncompressed_tagged = int64_t(compressed_tagged);
}
```

让我们尝试更改压缩方案以简化解压缩代码。

### 堆布局，v2

如果不是在4 GB的开头有基数，而是将基数放在*中间*，我们可以将压缩值视为**签署**从基极偏移 32 位。请注意，整个预留不再是 4 GB 对齐的，但基本保留是。

![Heap layout, base aligned to the middle](/\_img/pointer-compression/heap-layout-2.svg)

在此新布局中，压缩代码保持不变。

但是，解压缩代码会变得更好。符号扩展现在对于Smi和指针情况都很常见，唯一的分支是是否在指针情况下添加基数。

```cpp
int32_t compressed_tagged;

// Common code for both pointer and Smi cases
int64_t uncompressed_tagged = int64_t(compressed_tagged);
if (uncompressed_tagged & 1) {
  // pointer case
  uncompressed_tagged += base;
}
```

代码中分支的性能取决于 CPU 中的分支预测单元。我们认为，如果我们以无分支的方式实现解压缩，我们可以获得更好的性能。有了少量的位魔术，我们可以编写上面代码的无分支版本：

```cpp
int32_t compressed_tagged;

// Same code for both pointer and Smi cases
int64_t sign_extended_tagged = int64_t(compressed_tagged);
int64_t selector_mask = -(sign_extended_tagged & 1);
// Mask is 0 in case of Smi or all 1s in case of pointer
int64_t uncompressed_tagged =
    sign_extended_tagged + (base & selector_mask);
```

然后，我们决定从无分支实现开始。

## 性能演进

### 初始性能

我们衡量了以下方面的表现：[辛烷](https://v8.dev/blog/retiring-octane#the-genesis-of-octane)— 我们过去使用过的峰值性能基准测试。虽然我们不再专注于提高日常工作中的峰值性能，但我们也不想让峰值性能倒退，特别是对于像性能敏感这样敏感的事情。*所有指针*.Octane 仍然是这项任务的良好基准。

此图显示了 Octane 在优化和完善指针压缩实现时在 x64 架构上的得分。在图中，越高越好。红线是现有的全尺寸指针 x64 版本，而绿线是指针压缩版本。

![First round of Octane’s improvements](/\_img/pointer-compression/perf-octane-1.svg)

在第一个工作实现中，我们有大约35%的回归差距。

#### 颠簸 （1）， +7%

首先，我们通过比较无分支减压和分支解压来验证我们的“无分支更快”假设。事实证明，我们的假设是错误的，分支版本在x64上的速度提高了7%。这是一个相当大的区别！

让我们看一下 x64 程序集。

：：：表包装器

<!-- markdownlint-disable no-space-in-code -->

|减压|无分支|分支|
|---------------|-------------------------|------------------------------|
|代码|` asm                  |  `阿斯姆\
|              |movsxlq r11，\[...]        |movsxlq r11，\[...]             \
|              |移动 r10，r11 |testb r11，0x1               \
|              |10，0x1 |兰特jz done                     \
|              |negq r10 |地址 r11，r13                \
|              |10，r13 |做：                       \
|              |地址 r11，r10 |                             |\
|              |`                     | `|
|总结|20 字节 |13 字节 |
|^^            ||执行的6条指令|执行3或4条指令
|^^            |没有分支|1 个分行|
|^^            |1 个附加寄存器|                             |

<!-- markdownlint-enable no-space-in-code -->

:::

**r13**这是一个专用的寄存器，用于基本值。请注意，无分支代码既更大，又需要更多的寄存器。

在Arm64上，我们观察到了相同的情况 - 分支版本在功能强大的CPU上显然更快（尽管两种情况下的代码大小相同）。

：：：表包装器

<!-- markdownlint-disable no-space-in-code -->

|减压|无分支|分支|
|---------------|-------------------------|------------------------------|
|代码|` asm                  |  `阿斯姆\
|              |ldur w6， \[...]           |ldur w6， \[...]                \
|              |sbfx x16， x6， #0， #1 |sxtw x6， w6                 \
|              |和 x16、x16、x26 |tbz w6， #0， #done           \
|              |添加 x6， x16， w6， sxtw |添加 x6， x26， x6             \
|              |                        |做：                       \
|              |`                     | `|
|总结|16 字节 |16 字节 |
|^^            ||执行的4条指令|执行3或4条指令
|^^            |没有分支|1 个分行|
|^^            |1 个附加寄存器|                             |

<!-- markdownlint-enable no-space-in-code -->

:::

在低端Arm64设备上，我们观察到两个方向的性能差异几乎没有差异。

我们的结论是：现代CPU中的分支预测器非常好，代码大小（特别是执行路径长度）对性能的影响更大。

#### 颠簸 （2）， +2%

[涡轮风扇](https://v8.dev/docs/turbofan)是V8的优化编译器，围绕一个名为“节点之海”的概念构建。简而言之，每个操作都表示为图形中的节点（请参阅更详细的版本[在这篇博客文章中](https://v8.dev/blog/turbofan-jit)).这些节点具有各种依赖项，包括数据流和控制流。

有两个操作对于指针压缩至关重要：加载和存储，因为它们将 V8 堆与管道的其余部分连接起来。如果我们每次从堆中加载压缩值时都要解压缩，并在存储之前对其进行压缩，那么管道就可以像在完整指针模式下一样继续工作。因此，我们在节点图中添加了新的显式值操作 - 解压缩和压缩。

在某些情况下，实际上不需要减压。例如，如果从某个位置加载压缩值，然后才存储到新位置。

为了优化不必要的操作，我们在 TurboFan 中实施了新的“减压消除”阶段。它的工作是消除紧接着按压的解压。由于这些节点可能不直接相邻，它还会尝试通过图形传播解压缩，希望遇到压缩并消除它们。这使 Octane 的分数提高了 2%。

#### 颠簸 （3）， +2%

当我们查看生成的代码时，我们注意到刚刚加载的值的解压缩产生的代码有点太冗长了：

```asm
movl rax, <mem>   // load
movlsxlq rax, rax // sign extend
```

一旦我们修复了它来签名，就直接扩展从内存加载的值：

```asm
movlsxlq rax, <mem>
```

所以又改善了2%。

#### 颠簸 （4）， +11%

TurboFan优化阶段通过在图形上使用模式匹配来工作：一旦子图与某个模式匹配，它就会被语义上等效（但更好）的子图或指令所取代。

不成功的查找匹配项的尝试不是显式失败。图中存在显式的解压缩/压缩操作，导致以前成功的模式匹配尝试不再成功，从而导致优化静默失败。

“中断”优化的一个例子是[分配预处理](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43823.pdf).一旦我们更新了模式匹配，以了解新的压缩/解压缩节点，我们又提高了11%。

### 进一步改进

![Second round of Octane’s improvements](/\_img/pointer-compression/perf-octane-2.svg)

#### 凸块 （5）， +0.5%

在 TurboFan 中实施减压消除时，我们学到了很多东西。显式解压缩/压缩节点方法具有以下属性：

优点：

*   此类操作的显式性允许我们通过对子图进行规范模式匹配来优化不必要的解压缩。

但是，当我们继续实施时，我们发现了缺点：

*   由于新的内部价值表示，可能的转换操作的组合爆炸变得无法控制。除了现有的一组表示（标记的Smi，标记的指针，标记的指针，标记的任意，word8，word16，word32，word64，float32，float64，float64，simd128）之外，我们现在可以压缩指针，压缩Smi和压缩任何（我们可以是指针或Smi的压缩值）。
*   一些基于图形模式匹配的现有优化静默地没有触发，这导致了这里和那里的回归。虽然我们发现并修复了其中的一些问题，但TurboFan的复杂性继续增加。
*   寄存器分配器对图中的节点数量越来越不满意，并且经常生成错误的代码。
*   较大的节点图减慢了 TurboFan 优化阶段的速度，并增加了编译期间的内存消耗。

我们决定退后一步，想出一种更简单的方法来支持TurboFan中的指针压缩。 新方法是删除压缩指针/ Smi / Any表示，并使所有显式压缩/解压缩节点隐式在存储和加载中，并假设我们总是在加载之前解压缩，并在存储之前压缩。

我们还在TurboFan中增加了一个新阶段，将取代“减压消除”阶段。这个新阶段将认识到我们实际上不需要压缩或解压缩并相应地更新加载和存储。这种方法大大降低了 TurboFan 中指针压缩支持的复杂性，并提高了生成代码的质量。

新的实现与初始版本一样有效，并又提高了0.5%。

#### 颠簸 （6）， +2.5%

我们接近性能平价，但差距仍然存在。我们必须想出更新鲜的想法。其中之一是：如果我们确保任何处理Smi值的代码永远不会“查看”上面的32位，该怎么办？

让我们记住解压缩实现：

```cpp
// Old decompression implementation
int64_t uncompressed_tagged = int64_t(compressed_tagged);
if (uncompressed_tagged & 1) {
  // pointer case
  uncompressed_tagged += base;
}
```

如果忽略Smi的上限32位，我们可以假设它们是未定义的。然后，我们可以避免指针和Smi案例之间的特殊外壳，并在解压时无条件添加底座，即使对于Smis也是如此！我们将这种方法称为“Smi-corrupting”。

```cpp
// New decompression implementation
int64_t uncompressed_tagged = base + int64_t(compressed_tagged);
```

此外，由于我们不再关心符号扩展Smi，因此此更改允许我们返回到堆布局v1。这是基数指向4GB预留开头的那个。

<figure>
  <img src="/_img/pointer-compression/heap-layout-1.svg" width="827" height="323" alt="" loading="lazy">
  <figcaption>Heap layout, base aligned to start</figcaption>
</figure>

就解压缩代码而言，它将符号扩展操作更改为零扩展，这同样便宜。但是，这简化了运行时（C++）方面的工作。例如，地址空间区域预留代码（请参阅[一些实现细节](#some-implementation-details)部分）。

下面是用于比较的程序集代码：

：：：表包装器

<!-- markdownlint-disable no-space-in-code -->

|减压|分支|破坏|
|---------------|------------------------------|------------------------------|
|代码|` asm                       |  `阿斯姆\
|              |movsxlq r11，\[...]             |movl r11，\[rax+0x13]         \
|              |testb r11，0x1 |地址 r11，r13                \
|              |jz done |                             |\
|              |地址 r11，r13 |                             |\
|              |完成：|                             |\
|              |`                          | `|
|总结|13 字节 |7 字节 |
|^^            ||执行3或4条指令2 条指令|执行
|^^            |1 个分行|没有分支|

<!-- markdownlint-enable no-space-in-code -->

:::

因此，我们将 V8 中的所有 Smi 使用代码段调整为新的压缩方案，这又提高了 2.5%。

### 剩余缺口

剩余的性能差距可以通过针对 64 位构建的两项优化来解释，由于与指针压缩基本不兼容，我们不得不禁用这两个优化。

![Final round of Octane’s improvements](/\_img/pointer-compression/perf-octane-3.svg)

#### 32 位 SMI 优化 （7），-1%

让我们回想一下 Smis 在 64 位架构上的全指针模式下的样子。

            |----- 32 bits -----|----- 32 bits -----|
    Smi:    |____int32_value____|0000000000000000000|

32 位 Smi 具有以下优点：

*   它可以表示更大范围的整数，而无需将它们框入数字对象;和
*   这样的形状在读取/写入时提供对 32 位值的直接访问。

这种优化无法使用指针压缩来完成，因为 32 位压缩指针中没有空间，因为具有将指针与 Smis 区分开来的位。如果我们在全指针 64 位版本中禁用 32 位 smis，我们会看到 Octane 分数的回归率为 1%。

#### 双字段拆箱 （8）， -3%

此优化尝试在某些假设下将浮点值直接存储在对象的字段中。这样做的目的是减少对象分配的数量，甚至比Smis单独使用的数量还要多。

想象一下下面的JavaScript代码：

```js
function Point(x, y) {
  this.x = x;
  this.y = y;
}
const p = new Point(3.1, 5.3);
```

一般来说，如果我们看一下对象p在内存中的样子，我们会看到这样的东西：

![Object p in memory](/\_img/pointer-compression/heap-point-1.svg)

您可以在 中阅读有关隐藏类和属性以及元素后备存储区的详细信息[本文](https://v8.dev/blog/fast-properties).

在 64 位体系结构上，双精度值的大小与指针相同。因此，如果我们假设 Point 的字段始终包含数字值，则可以将它们直接存储在对象字段中。

![](/\_img/pointer-compression/heap-point-2.svg)

如果假设对某个字段无效，请在执行此行后说：

```js
const q = new Point(2, 'ab');
```

则 y 属性的数字值必须以盒装形式存储。此外，如果在某个地方存在依赖于此假设的推测优化代码，则必须不再使用它，并且必须将其丢弃（取消优化）。这种“字段类型”泛化的原因是尽量减少从同一构造函数创建的对象形状的数量，这反过来又是更稳定的性能所必需的。

![Objects p and q in memory](/\_img/pointer-compression/heap-point-3.svg)

如果应用，双字段取消装箱具有以下优点：

*   通过对象指针提供对浮点数据的直接访问，避免通过数字对象进行额外的反引用;和
*   允许我们生成更小、更快的优化代码，用于执行大量双字段访问的紧密循环（例如在数字运算应用程序中）

启用指针压缩后，双精度值将不再适合压缩字段。但是，将来我们可能会针对指针压缩调整此优化。

请注意，即使没有这种双字段解装优化（以与指针压缩兼容的方式），也可以通过将数据存储在 Float64 TypedArrays 中，或者甚至通过使用[瓦斯姆](https://webassembly.github.io/spec/core/).

#### 更多改进 （9）， 1%

最后，对 TurboFan 中的减压消除优化进行了一些微调，性能又提高了 1%。

## 一些实现细节

为了简化指针压缩到现有代码中的集成，我们决定在每次加载时解压缩值，并在每个存储上压缩它们。因此，仅更改标记值的存储格式，同时保持执行格式不变。

### 本机代码端

为了能够在需要解压缩时生成有效的代码，基值必须始终可用。幸运的是，V8 已经有一个专用的寄存器，它始终指向一个“根表”，其中包含对 JavaScript 和 V8 内部对象的引用，这些对象必须始终可用（例如，未定义、空、真、假等等）。此寄存器称为“根寄存器”，用于生成较小的和[可共享的内置代码](https://v8.dev/blog/embedded-builtins).

因此，我们将根表放入 V8 堆预留区域，因此根寄存器可用于这两个目的 - 作为根指针和作为解压缩的基值。

### C++侧

V8 运行时通过C++类访问 V8 堆中的对象，从而提供堆中存储的数据的便捷视图。请注意，V8 对象是相当[荚](https://en.wikipedia.org/wiki/Passive_data_structure)-像结构比C++物体。帮助程序“view”类仅包含一个具有相应标记值的uintptr_t字段。由于视图类是字大小的，我们可以按值传递它们，开销为零（这在很大程度上要归功于现代C++编译器）。

下面是一个辅助类的伪示例：

```cpp
// Hidden class
class Map {
 public:
  …
  inline DescriptorArray instance_descriptors() const;
  …
  // The actual tagged pointer value stored in the Map view object.
  const uintptr_t ptr_;
};

DescriptorArray Map::instance_descriptors() const {
  uintptr_t field_address =
      FieldAddress(ptr_, kInstanceDescriptorsOffset);

  uintptr_t da = *reinterpret_cast<uintptr_t*>(field_address);
  return DescriptorArray(da);
}
```

为了最大限度地减少首次运行指针压缩版本所需的更改次数，我们将解压缩所需的基值的计算集成到 getter 中。

```cpp
inline uintptr_t GetBaseForPointerCompression(uintptr_t address) {
  // Round address down to 4 GB
  const uintptr_t kBaseAlignment = 1 << 32;
  return address & -kBaseAlignment;
}

DescriptorArray Map::instance_descriptors() const {
  uintptr_t field_address =
      FieldAddress(ptr_, kInstanceDescriptorsOffset);

  uint32_t compressed_da = *reinterpret_cast<uint32_t*>(field_address);

  uintptr_t base = GetBaseForPointerCompression(ptr_);
  uintptr_t da = base + compressed_da;
  return DescriptorArray(da);
}
```

性能测量证实，计算每个负载中的基数会损害性能。原因是C++编译器不知道 GetBaseForPointerCompression（） 调用的结果对于 V8 堆中的任何地址都是相同的，因此编译器无法合并基值的计算。鉴于代码由几条指令和一个64位常量组成，这会导致明显的代码膨胀。

为了解决这个问题，我们重用了 V8 实例指针作为解压缩的基础（请记住堆布局中的 V8 实例数据）。此指针通常在运行时函数中可用，因此我们通过要求 V8 实例指针简化了 getters 代码，并恢复了回归：

```cpp
DescriptorArray Map::instance_descriptors(const Isolate* isolate) const {
  uintptr_t field_address =
      FieldAddress(ptr_, kInstanceDescriptorsOffset);

  uint32_t compressed_da = *reinterpret_cast<uint32_t*>(field_address);

  // No rounding is needed since the Isolate pointer is already the base.
  uintptr_t base = reinterpret_cast<uintptr_t>(isolate);
  uintptr_t da = DecompressTagged(base, compressed_value);
  return DescriptorArray(da);
}
```

## 结果

让我们来看看指针压缩的最终数字！对于这些结果，我们使用在本博客文章开头介绍的相同浏览测试。提醒一下，他们正在浏览我们发现代表现实世界网站使用情况的用户故事。

在其中，我们观察到指针压缩减少了**V8 堆大小高达 43%**!反过来，它减少了**Chrome 的渲染器进程内存高达 20%**在桌面上。

![Memory savings when browsing in Windows 10](/\_img/pointer-compression/v8-heap-memory.svg)

需要注意的另一件重要事情是，并非每个网站的改进量相同。例如，Facebook上的V8堆内存曾经比纽约时报大，但使用指针压缩实际上是相反的。这种差异可以通过以下事实来解释：某些网站比其他网站具有更多的标记值。

除了这些内存改进之外，我们还看到了实际性能的改进。在真实的网站上，我们利用更少的CPU和垃圾回收器时间！

![Improvements in CPU and garbage collection time](/\_img/pointer-compression/performance-improvements.svg)

## 结论

到达这里的旅程没有玫瑰床，但值得我们花时间。[300 多个提交](https://github.com/v8/v8/search?o=desc\&q=repo%3Av8%2Fv8+%22%5Bptr-compr%5D%22\&s=committer-date\&type=Commits)后来，带有指针压缩的 V8 使用与运行 32 位应用程序一样多的内存，同时具有 64 位应用程序的性能。

我们一直期待着改进，并在我们的管道中有以下相关任务：

*   提高生成的汇编代码的质量。我们知道，在某些情况下，我们可以生成更少的代码，从而提高性能。
*   解决相关的性能回归问题，包括一种允许以指针压缩友好的方式再次取消对双字段进行装箱的机制。
*   探索支持 8 到 16 GB 范围内更大堆的想法。
