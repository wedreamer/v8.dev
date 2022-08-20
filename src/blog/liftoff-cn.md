***

title： 'Liftoff： 一个新的基准编译器，用于 V8 中的 WebAssembly'
作者：《Clemens Backes， WebAssembly Compilation Maestro》
化身：

*   “克莱门斯-巴克尔斯”
    日期： 2018-08-20 15：45：12
    标签：
*   WebAssembly
*   内部
    描述： 'Liftoff 是 WebAssembly 的一个新的基准编译器，在 V8 v6.9 中发布。
    推文：“1031538167617413120”

***

V8[版本6.9](/blog/v8-release-69)包括Liftoff，一个新的WebAssembly基线编译器。现在，默认情况下，在桌面系统上启用 Liftoff。本文详细介绍了添加另一个编译层的动机，并描述了 Liftoff 的实现和性能。

<figure>
  <img src="/_img/v8-liftoff.svg" width="256" height="256" alt="" loading="lazy">
  <figcaption>Logo for Liftoff, V8’s WebAssembly baseline compiler</figcaption>
</figure>

自 WebAssembly 以来[推出](/blog/v8-release-57)一年多以前，网络上的采用率一直在稳步增长。针对WebAssembly的大型应用程序已经开始出现。例如，Epic的[禅园基准](https://s3.amazonaws.com/mozilla-games/ZenGarden/EpicZenGarden.html)包含一个 39.5 MB 的 WebAssembly 二进制文件，以及[欧特克](https://web.autocad.com/)以 36.8 MB 二进制文件的形式提供。由于编译时间在二进制大小上基本上是线性的，因此这些应用程序需要相当长的时间来启动。在许多计算机上，它超过30秒，这并不能提供出色的用户体验。

但是，如果类似的JS应用程序启动得更快，为什么启动WebAssembly应用程序需要这么长时间呢？原因是WebAssembly承诺提供*可预测的性能*，因此，一旦应用程序运行，您可以确保始终如一地满足性能目标（例如，每秒渲染60帧，没有音频延迟或伪像等）。为了实现这一点，WebAssembly代码被编译*超前*在 V8 中，避免了实时编译器引入的任何编译暂停，这可能导致应用程序中出现明显的卡顿。

## 现有的编译管道（TurboFan）

V8 编译 WebAssembly 的方法依赖于*涡轮风扇*，我们为JavaScript和asm设计的优化编译器.js。TurboFan是一个功能强大的编译器，具有基于图形的编译器*中间表示 （IR）*适用于高级优化，如强度降低、内联、代码运动、指令组合和复杂的寄存器分配。TurboFan的设计支持很晚才进入管道，更接近机器代码，这绕过了支持JavaScript编译所需的许多阶段。通过设计，将WebAssembly代码转换为TurboFan的IR（包括[*SSA-结构*](https://en.wikipedia.org/wiki/Static_single_assignment_form)）在一次简单的单次传递中是非常有效的，部分原因是WebAssembly的结构化控制流。然而，编译过程的后端仍然消耗大量的时间和内存。

## 新的编译管道（Liftoff）

Liftoff的目标是通过尽可能快地生成代码来减少基于WebAssembly的应用程序的启动时间。代码质量是次要的，因为无论如何，热代码最终都会用TurboFan重新编译。Liftoff 避免了构建 IR 的时间和内存开销，并在 WebAssembly 函数的字节码上的一次传递中生成机器代码。

![The Liftoff compilation pipeline is much simpler compared to the TurboFan compilation pipeline.](/\_img/liftoff/pipeline.svg)

从上图可以明显看出，Liftoff应该能够比TurboFan更快地生成代码，因为管道仅由两个阶段组成。事实上，*函数体解码器*对原始WebAssembly字节进行单次传递，并通过回调与后续阶段进行交互，因此*代码生成*执行*解码和验证时*函数体。与WebAssembly一起*[流式处理接口](/blog/v8-release-65)*，这允许 V8 在通过网络下载时将 WebAssembly 代码编译为机器代码。

### 在 Liftoff 中生成代码

Liftoff是一个简单的代码生成器，而且速度很快。它只对函数的操作码执行一次传递，为每个操作码生成代码，一次一个。对于像算术这样的简单操作码，这通常是单个机器指令，但对于其他操作码（如调用）来说可能更多。Liftoff 维护有关操作数堆栈的元数据，以便了解每个操作的输入当前存储的位置。这*虚拟堆栈*仅在编译期间存在。WebAssembly的结构化控制流和验证规则保证了这些输入的位置可以静态确定。因此，不需要将操作数推送和弹出到的实际运行时堆栈。在执行期间，虚拟堆栈上的每个值要么保存在寄存器中，要么溢出到该函数的物理堆栈帧中。对于小整数常量（由`i32.const`），Liftoff 仅记录虚拟堆栈中常量的值，不生成任何代码。仅当后续操作使用常量时，才会发出常量或将其与操作组合，例如通过直接发出`addl <reg>, <const>`x64 上的说明。这样可以避免将该常量加载到寄存器中，从而产生更好的代码。

让我们通过一个非常简单的函数来看看Liftoff如何为此生成代码。

![](/\_img/liftoff/example-1.svg)

此示例函数采用两个参数并返回其总和。当 Liftoff 解码此函数的字节时，它首先根据 WebAssembly 函数的调用约定初始化局部变量的内部状态。对于 x64，V8 的调用约定传递寄存器中的两个参数*拉克斯*和*断续器*.

为`get_local`指令，Liftoff不会生成任何代码，而只是更新其内部状态以反映这些寄存器值现在被推送到虚拟堆栈上。这`i32.add`然后，指令弹出两个寄存器，并为结果值选择一个寄存器。我们不能使用任何输入寄存器来获取结果，因为两个寄存器仍然出现在堆栈上以保存局部变量。覆盖它们将更改稍后返回的值`get_local`指令。因此，在这种情况下，Liftoff选择了一个免费的寄存器*rcx*，并生成*拉克斯*和*断续器*进入该寄存器。*rcx*然后被推送到虚拟堆栈上。

在`i32.add`指令，函数体完成，所以 Liftoff 必须组装函数返回。由于我们的示例函数有一个返回值，因此验证要求函数体末尾的虚拟堆栈上必须正好有一个值。因此，Liftoff 会生成移动保存在 中的返回值的代码*rcx*进入正确的退货登记簿*拉克斯*，然后从函数返回。

为简单起见，上面的示例不包含任何块（`if`,`loop`...)或分支。WebAssembly 中的块引入了控件合并，因为代码可以分支到任何父块，并且可以跳过 if 块。可以从不同的堆栈状态访问这些合并点。但是，以下代码必须假定特定的堆栈状态才能生成代码。因此，Liftoff将虚拟堆栈的当前状态快照为新块后面的代码将假定的状态（即，当返回到*控制级别*我们目前所处的位置）。然后，新块将继续保持当前活动状态，可能会更改堆栈值或局部变量的存储位置：一些可能溢出到堆栈或保存在其他寄存器中。当分支到另一个块或结束一个块（这与分支到父块相同）时，Liftoff必须生成使当前状态适应该点的预期状态的代码，以便为目标发出的代码，我们分支在它期望的位置找到正确的值。验证可保证当前虚拟堆栈的高度与预期状态的高度相匹配，因此 Liftoff 只需生成代码即可在寄存器和/或物理堆栈帧之间随机排列值，如下所示。

让我们看一个例子。

![](/\_img/liftoff/example-2.svg)

上面的示例假定一个虚拟堆栈在操作数堆栈上具有两个值。在开始新块之前，虚拟堆栈上的最高值将作为参数弹出到`if`指令。剩余的堆栈值需要放在另一个寄存器中，因为它当前正在隐藏第一个参数，但是当分支回此状态时，我们可能需要为堆栈值和参数保留两个不同的值。在这种情况下，Liftoff 选择将其重复数据删除到*rcx*注册。然后对此状态进行快照，并在块内修改活动状态。在块的末尾，我们隐式分支回父块，因此我们通过移动寄存器将当前状态合并到快照中。*断续器*到*rcx*和重新加载寄存器*断续器*从堆栈帧。

### 从升降到涡轮风扇的分层

借助Liftoff和TurboFan，V8现在为WebAssembly提供了两个编译层：Liftoff作为快速启动的基准编译器，TurboFan作为优化编译器以获得最佳性能。这就提出了如何组合两个编译器以提供最佳整体用户体验的问题。

对于 JavaScript，V8 使用 Ignition 解释器和 TurboFan 编译器，并采用动态分层策略。每个函数首先在Ignition中执行，如果函数变热，TurboFan会将其编译为高度优化的机器代码。类似的方法也可以用于Liftoff，但这里的权衡有点不同：

1.  WebAssembly不需要类型反馈来生成快速代码。JavaScript从收集类型反馈中受益匪浅，而WebAssembly是静态类型的，因此引擎可以立即生成优化的代码。
2.  WebAssembly 代码应该运行*不出所料*快速，无需冗长的预热阶段。应用程序面向WebAssembly的原因之一是在Web上执行*具有可预测的高性能*.因此，我们既不能容忍运行次优代码太长时间，也不能接受执行期间的编译暂停。
3.  Ignition解释器的JavaScript的一个重要设计目标是通过根本不编译函数来减少内存使用量。然而，我们发现WebAssembly的解释器太慢了，无法实现可预测的快速性能目标。事实上，我们确实构建了这样一个解释器，但是由于比编译的代码慢20×或更慢，因此无论节省多少内存，它都仅用于调试。鉴于此，引擎无论如何都必须存储编译的代码;最后，它应该只存储最紧凑和最有效的代码，这是TurboFan优化的代码。

从这些约束中，我们得出结论，对于V8目前实现WebAssembly来说，动态分层并不是正确的权衡，因为它会增加代码大小并降低性能，并且不会在不确定的时间跨度内降低性能。相反，我们选择了一种策略*渴望分层*.在模块的 Liftoff 编译完成后，WebAssembly 引擎会立即启动后台线程，为模块生成优化的代码。这允许 V8 快速开始执行代码（在 Liftoff 完成后），但仍然能尽早获得性能最高的 TurboFan 代码。

下图显示了编译和执行的痕迹[EpicZenGarden基准测试](https://s3.amazonaws.com/mozilla-games/ZenGarden/EpicZenGarden.html).它表明，在Liftoff编译之后，我们可以实例化WebAssembly模块并开始执行它。TurboFan编译仍然需要几秒钟的时间，因此在分层期间，观察到的执行性能会逐渐增加，因为各个TurboFan功能一完成就会被使用。

![](/\_img/liftoff/tierup-liftoff-turbofan.png)

## 性能

有两个指标对于评估新的 Liftoff 编译器的性能很有趣。首先，我们要将编译速度（即生成代码的时间）与TurboFan进行比较。其次，我们要测量生成的代码的性能（即执行速度）。第一个度量值在这里更有趣，因为Liftoff的目标是通过尽快生成代码来减少启动时间。另一方面，生成的代码的性能应该仍然相当不错，因为该代码可能仍然在低端硬件上执行几秒钟甚至几分钟。

### 生成代码的性能

用于测量*编译器性能*本身，我们运行了许多基准测试，并使用跟踪来测量原始编译时间（见上图）。我们在 HP Z840 机器（2 个英特尔至强 E5-2690 @2.6GHz，24 个内核，48 个线程）和 Macbook Pro（英特尔酷睿 i7-4980HQ @2.8GHz，4 个内核，8 个线程）上运行这两个基准测试。请注意，Chrome 目前使用的后台线程不超过 10 个，因此 Z840 计算机的大多数内核都未使用。

我们执行三个基准测试：

1.  [**EpicZenGarden**](https://s3.amazonaws.com/mozilla-games/ZenGarden/EpicZenGarden.html)：在Epic框架上运行的ZenGarden演示
2.  [**坦克！**](https://webassembly.org/demo/)：Unity 引擎的演示
3.  [**欧特克**](https://web.autocad.com/)
4.  [**PSPDFKit**](https://pspdfkit.com/webassembly-benchmark/)

对于每个基准测试，我们使用跟踪输出来测量原始编译时间，如上所示。这个数字比基准测试本身报告的任何时间都更稳定，因为它不依赖于在主线程上计划的任务，也不包括不相关的工作，如创建实际的WebAssembly实例。

下图显示了这些基准测试的结果。每个基准测试执行了三次，我们报告平均编译时间。

![Code generation performance of Liftoff vs. TurboFan on a MacBook](/\_img/liftoff/performance-unity-macbook.svg)

![Code generation performance of Liftoff vs. TurboFan on a Z840](/\_img/liftoff/performance-unity-z840.svg)

正如预期的那样，Liftoff编译器在高端台式机工作站和MacBook上生成代码的速度要快得多。Liftoff对TurboFan的加速在功能较差的MacBook硬件上甚至更大。

### 生成代码的性能

尽管生成的代码的性能是次要目标，但我们希望在启动阶段保持高性能的用户体验，因为Liftoff代码可能会在TurboFan代码完成之前执行几秒钟。

为了测量 Liftoff 代码性能，我们关闭了层级以测量纯 Liftoff 执行。在此设置中，我们执行两个基准测试：

1.  **Unity 无外设基准测试**

    这是在 Unity 框架中运行的许多基准测试。它们是无头的，因此可以直接在d8 shell中执行。每个基准测试都会报告一个分数，该分数不一定与执行性能成正比，但足以比较性能。

2.  [**PSPDFKit**](https://pspdfkit.com/webassembly-benchmark/)

    此基准测试报告了对 pdf 文档执行不同操作所需的时间以及实例化 WebAssembly 模块（包括编译）所需的时间。

和以前一样，我们执行每个基准测试三次，并使用三次运行的平均值。由于基准之间记录的数字规模差异很大，因此我们报告*升降与涡轮风扇的相对性能*.值*+30%*这意味着Liftoff代码的运行速度比TurboFan慢30%。负数表示 Liftoff 执行速度更快。结果如下：

![Liftoff Performance on Unity](/\_img/liftoff/performance-unity-compile.svg)

在Unity上，Liftoff代码在台式机器上的平均执行速度比TurboFan代码慢50%左右，在MacBook上慢70%。有趣的是，有一种情况（曼德布洛特脚本）的Liftoff代码优于TurboFan代码。这可能是一个异常值，例如，TurboFan的寄存器分配器在热循环中表现不佳。我们正在调查是否可以改进TurboFan以更好地处理此案。

![Liftoff Performance on PSPDFKit](/\_img/liftoff/performance-pspdfkit-compile.svg)

在 PSPDFKit 基准测试中，Liftoff 代码的执行速度比优化代码慢 18-54%，而初始化则如预期的那样显著改进。这些数字表明，对于也通过JavaScript调用与浏览器交互的实际代码，未优化代码的性能损失通常低于计算密集型基准测试。

同样，请注意，对于这些数字，我们完全关闭了分层，因此我们只执行了Liftoff代码。在生产配置中，Liftoff代码将逐渐被TurboFan代码取代，使得Liftoff代码的较低性能仅持续很短的时间。

## 三. 今后的工作

在Liftoff首次推出后，我们正在努力进一步缩短启动时间，减少内存使用量，并将Liftoff的好处带给更多用户。特别是，我们正在努力改进以下几点：

1.  **Port Liftoff到arm和arm64，也可以在移动设备上使用它。**目前，Liftoff仅适用于英特尔平台（32位和64位），主要捕获桌面用例。为了吸引移动用户，我们将把Liftoff移植到更多的架构上。
2.  **为移动设备实施动态分层。**由于移动设备的可用内存往往比桌面系统少得多，因此我们需要针对这些设备调整分层策略。只需使用TurboFan重新编译所有功能，即可轻松地将保存所有代码所需的内存增加一倍，至少是暂时的（直到放弃Liftoff代码）。相反，我们正在尝试将延迟编译与Liftoff和TurboFan中热函数的动态分层相结合。
3.  **提高 Liftoff 代码生成的性能。**实现的第一次迭代很少是最好的迭代。有几件事可以调整，以进一步加快Liftoff的编译速度。这将在下一个版本中逐渐发生。
4.  **提高 Liftoff 代码的性能。**除了编译器本身，生成的代码的大小和速度也可以提高。这也将在下一个版本中逐步发生。

## 结论

V8 现在包含 Liftoff，这是 WebAssembly 的新基准编译器。Liftoff通过简单快速的代码生成器大大减少了WebAssembly应用程序的启动时间。在桌面系统上，通过使用 TurboFan 在后台重新编译所有代码，V8 仍能达到最大峰值性能。默认情况下，在 V8 v6.9 （Chrome 69） 中启用 Liftoff，并且可以使用`--liftoff`/`--no-liftoff`和`chrome://flags/#enable-webassembly-baseline`分别是每个标志。
