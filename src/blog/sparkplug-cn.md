***

title： 'Sparkplug — 一个非优化的 JavaScript 编译器'
作者： '[莱谢克·斯维尔斯基](https://twitter.com/leszekswirski)——也许不是最亮的火花，但至少是最快的火花。
化身：

*   leszek-swirski
    日期： 2021-05-27
    标签：
*   JavaScript
    extra_links：
*   href： https://fonts.googleapis.com/css?family=Gloria+Hallelujah\&display=swap
    rel： 样式表
    描述： “在 V8 v9.1 中，我们使用 Sparkplug 将 V8 性能提高了 5–15%：这是一个新的、非优化的 JavaScript 编译器。
    推文：“1397945205198835719”

***

<!-- markdownlint-capture -->

<!-- markdownlint-disable no-inline-html -->

<style>
  svg {
    --other-frame-bg: rgb(200 200 200 / 20%);
    --machine-frame-bg: rgb(200 200 200 / 50%);
    --js-frame-bg: rgb(212 205 100 / 60%);
    --interpreter-frame-bg: rgb(215 137 218 / 50%);
    --sparkplug-frame-bg: rgb(235 163 104 / 50%);
  }
  svg text {
    font-family: Gloria Hallelujah, cursive;
  }
  .flipped .frame {
    transform: scale(1, -1);
  }
  .flipped .frame text {
    transform:scale(1, -1);
  }
</style>

<!-- markdownlint-restore -->

编写一个高性能的JavaScript引擎需要的不仅仅是拥有一个像TurboFan这样的高度优化的编译器。特别是对于短期会话，例如加载网站或命令行工具，在优化编译器有机会开始优化之前，还有很多工作要做，更不用说有时间生成优化代码了。

这就是为什么自 2016 年以来，我们从跟踪合成基准（如 Octane）转向测量的原因。[实际性能](/blog/real-world-performance)，以及为什么从那时起，我们在优化编译器之外一直在努力提高 JavaScript 的性能。这意味着在解析器、流式处理、我们的对象模型、垃圾回收器中的并发性、缓存编译代码方面的工作......让我们说我们从来没有感到无聊。

然而，当我们转向提高实际初始JavaScript执行的性能时，我们在优化解释器时开始遇到限制。V8的解释器高度优化且速度非常快，但解释器具有我们无法摆脱的固有开销;诸如字节码解码开销或调度开销之类的东西，它们是解释器功能的固有部分。

使用我们当前的双编译器模型，我们无法更快地升级到优化的代码;我们可以（并且正在）努力使优化更快，但在某些时候，您只能通过删除优化通道来加快速度，这会降低峰值性能。更糟糕的是，我们无法更早地开始优化，因为我们还没有稳定的物体形状反馈。

进入Sparkplug：我们新的非优化JavaScript编译器，我们将与V8 v9.1一起发布，它坐落在Ignition解释器和TurboFan优化编译器之间。

![The new compiler pipeline](/\_svg/sparkplug/pipeline.svg)

## 快速编译器

Sparkplug旨在快速编译。非常快。如此之快，我们几乎可以随时编译，这使我们能够比 TurboFan 代码更积极地升级到 Sparkplug 代码。

有几个技巧可以使Sparkplug编译器变得快速。首先，它作弊;它编译的函数已经编译成字节码，字节码编译器已经完成了大部分的艰苦工作，比如变量解析，弄清楚括号是否真的是箭头函数，去糖解构语句等等。Sparkplug从字节码编译，而不是从JavaScript源代码编译，因此不必担心任何这些。

第二个技巧是Sparkplug不会像大多数编译器那样生成任何中间表示（IR）。相反，Sparkplug在字节码上的单个线性传递中直接编译为机器代码，发出与该字节码的执行相匹配的代码。事实上，整个编译器是一个[`switch`陈述](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/baseline/baseline-compiler.cc;l=465;drc=55cbb2ce3be503d9096688b72d5af0e40a9e598b)在[`for`圈](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/baseline/baseline-compiler.cc;l=290;drc=9013bf7765d7febaa58224542782307fa952ac14)，调度到固定的每字节码机器码生成功能。

```cpp
// The Sparkplug compiler (abridged).
for (; !iterator.done(); iterator.Advance()) {
  VisitSingleBytecode();
}
```

缺少 IR 意味着编译器的优化机会有限，除了非常局部的窥视孔优化之外。这也意味着我们必须将整个实现分别移植到我们支持的每个架构中，因为没有中间的独立于架构的阶段。但是，事实证明，这两者都不是问题：快速编译器是一个简单的编译器，因此代码很容易移植;而Sparkplug不需要进行大量的优化，因为我们在以后的管道中有一个很棒的优化编译器。

：：： 备注
从技术上讲，我们目前对字节码进行两次传递 - 一次用于发现循环，另一次用于生成实际代码。不过，我们计划最终摆脱第一个。
:::

## 解释器兼容框架

向现有的成熟 JavaScript VM 添加新的编译器是一项艰巨的任务。除了标准执行之外，还有各种各样的东西需要支持。V8 有一个调试器，一个堆栈遍历 CPU 分析器，有异常的堆栈跟踪，集成到分层中，堆栈上替换为热循环的优化代码...这是很多。

Sparkplug做了一个巧妙的技巧，简化了大多数这些问题，即它保持了“解释器兼容的堆栈帧”。

让我们倒退一点。堆栈帧是代码执行存储函数状态的方式;每当您调用新函数时，它都会为该函数的局部变量创建一个新的堆栈帧。堆栈帧由帧指针（标记其开始）和堆栈指针（标记其结束）定义：

![A stack frame, with stack and frame pointers](/\_svg/sparkplug/basic-frame.svg)

：：： 备注

<!-- markdownlint-capture -->

<!-- markdownlint-disable no-inline-html -->

在这一点上，大约一半的人会尖叫，说“这张图没有意义，堆栈显然向相反的方向增长！别担心，我为你做了一个按钮：<button id="flipStacksButton">我认为堆栈向上增长</button>

<script>
  const flipStacksButton = document.getElementById('flipStacksButton');
  let stacksAreFlipped = Math.random() < 0.5;
  function updateStacks() {
    if (stacksAreFlipped) {
      document.body.classList.add('flipped');
      flipStacksButton.textContent = 'I think stacks grow downwards';
    } else {
      document.body.classList.remove('flipped');
      flipStacksButton.textContent = 'I think stacks grow upwards';
    }
  }
  updateStacks();
  flipStacksButton.onclick = () => {
    stacksAreFlipped = !stacksAreFlipped;
    updateStacks();
  };
</script>

<!-- markdownlint-restore -->

:::

当调用函数时，返回地址被推送到堆栈;当它返回时，函数会弹出它，以了解返回到的位置。然后，当该函数创建新帧时，它会将旧帧指针保存在堆栈上，并将新帧指针设置为其自己的堆栈帧的开头。因此，堆栈具有一系列帧指针，每个帧指针标记指向前一帧的帧的开始：

![Stack frames for multiple calls](/\_svg/sparkplug/machine-frame.svg)

：：： 备注
严格来说，这只是生成的代码所遵循的约定，而不是必需的。不过，这是一个非常普遍的。它唯一真正被破坏的时候是当堆栈帧被完全省略时，或者当调试侧表可以用来遍历堆栈帧时。
:::

这是所有类型函数的一般堆栈布局;然后有关于如何传递参数以及函数如何在其框架中存储值的约定。在 V8 中，我们有 JavaScript 帧的约定，即推送参数（包括接收器）[以相反的顺序](/blog/adaptor-frame)在调用函数之前在堆栈上，并且堆栈上的前几个插槽是：正在调用的当前函数;调用它的上下文;以及传递的参数数。这是我们的“标准”JS框架布局：

![A V8 JavaScript stack frame](/\_svg/sparkplug/js-frame.svg)

此 JS 调用约定在优化帧和解释帧之间共享，例如，它允许我们在调试器的性能面板中分析代码时以最小的开销遍历堆栈。

在Ignition解释器的情况下，约定变得更加明确。Ignition是一个基于寄存器的解释器，这意味着有虚拟寄存器（不要与机器寄存器混淆！）存储解释器的当前状态 - 这包括JavaScript函数局部变量（var/let/const声明）和临时值。这些寄存器存储在解释器的堆栈帧上，以及指向正在执行的字节码数组的指针，以及该数组中当前字节码的偏移量：

![A V8 interpreter stack frame](/\_svg/sparkplug/interpreter-frame.svg)

Sparkplug有意创建并维护与解释器框架相匹配的框架布局;每当解释器存储寄存器值时，Sparkplug也会存储一个寄存器值。它这样做有几个原因：

1.  它简化了Sparkplug编译;Sparkplug可以只镜像解释器的行为，而不必保留从解释器寄存器到Sparkplug状态的某种映射。
2.  它还加快了编译速度，因为字节码编译器已经完成了寄存器分配的艰苦工作。
3.  它使与系统其余部分的集成几乎微不足道;调试器，探查器，异常堆栈展开，堆栈跟踪打印，所有这些操作都做堆栈演练以发现当前执行函数堆栈是什么，所有这些操作都继续使用Sparkplug几乎不变，因为就它们而言，它们所拥有的只是一个解释器帧。
4.  它使堆栈上替换（OSR）变得微不足道。OSR是指在执行时替换当前正在执行的函数;目前，当解释函数位于热循环中（其中它分层到该循环的优化代码）以及优化的代码取消优化（其中它向下分层并继续在解释器中执行函数）时，就会发生这种情况。使用Sparkplug帧镜像解释器帧，任何适用于解释器的OSR逻辑都将适用于Sparkplug;更好的是，我们可以在解释器和Sparkplug代码之间切换，几乎为零帧转换开销。

我们对解释器堆栈帧进行了一个很小的更改，即在Sparkplug代码执行期间，我们不会使字节码偏移量保持最新。相反，我们存储从Sparkplug代码地址范围到相应字节码偏移量的双向映射;一个相对简单的编码映射，因为Sparkplug代码直接从字节码的线性游走中发出。每当堆栈帧访问想要知道Sparkplug帧的“字节码偏移量”时，我们都会在此映射中查找当前正在执行的指令并返回相应的字节码偏移量。类似地，每当我们想要OSR从解释器到Sparkplug时，我们可以在映射中查找当前的字节码偏移量，并跳转到相应的Sparkplug指令。

您可能会注意到，我们现在在堆栈帧上有一个未使用的插槽，字节码偏移量将在那里;一个我们无法摆脱的，因为我们希望保持堆栈的其余部分不变。我们重新利用这个堆栈插槽来缓存当前正在执行的函数的“反馈向量”;这是存储对象形状数据的向量，大多数操作都需要加载。我们所要做的就是在OSR上小心一点，以确保我们交换正确的字节码偏移量，或者此插槽的正确反馈向量。

因此，火花塞堆栈框架是：

![A V8 Sparkplug stack frame](/\_svg/sparkplug/sparkplug-frame.svg)

## 遵从内置

Sparkplug实际上很少生成自己的代码。JavaScript语义很复杂，即使最简单的操作也需要大量的代码。强制 Sparkplug 在每次编译时以内联方式重新生成此代码是不好的，原因有多种：

1.  它将从需要生成的大量代码中显着增加编译时间，
2.  这将增加Sparkplug代码的内存消耗，并且
3.  我们必须为Sparkplug的一堆JavaScript功能重新实现代码生成，这可能意味着更多的错误和更大的安全面。

因此，大多数Sparkplug代码只是调用“内置”，即嵌入二进制文件中的机器代码的小片段，而不是所有这些，来完成实际的脏工作。这些内置代码要么与解释器使用的内置项相同，要么至少与解释器的字节码处理程序共享其大部分代码。

实际上，Sparkplug代码基本上只是内置调用和控制流：

你现在可能会想，“那么，这一切有什么意义呢？Sparkplug不是在做和口译员一样的工作吗？“——你不会完全错的。在许多方面，Sparkplug“只是”解释器执行的序列化，调用相同的内置内容并保持相同的堆栈帧。然而，即使只是这样也是值得的，因为它消除了（或者更确切地说，预编译）那些不可移动的解释器开销，如操作数解码和下一个字节码调度。

事实证明，解释器会破坏许多CPU优化：解释器从内存中动态读取静态操作数，迫使CPU停止或推测值可能是什么;调度到下一个字节码需要成功的分支预测才能保持性能，即使推测和预测是正确的，您仍然必须执行所有解码和调度代码，并且您仍然在各种缓冲区和缓存中占用了宝贵的空间。CPU本身就是一个解释器，尽管是机器代码的解释器。从这个角度来看，Sparkplug是从Ignition字节码到CPU字节码的“转译器”，将你的函数从在“模拟器”中运行到运行“本机”。

## 性能

那么，Sparkplug在现实生活中的效果如何呢？我们在几个性能机器人上运行了Chrome 91，带有和不带有Sparkplug，以查看其影响。

剧透警告：我们非常高兴。

：：： 备注
以下基准测试列出了运行各种操作系统的各种机器人。尽管操作系统在机器人的名称中很突出，但我们认为它实际上对结果并没有太大影响。相反，不同的机器也有不同的CPU和内存配置，我们认为这是差异的大多数来源。
:::

# 速度计

[速度计](https://browserbench.org/Speedometer2.0/)是一个基准测试，它试图通过使用几个流行的框架构建TODO列表跟踪Web应用程序，并在添加和删除TODO时对该应用程序的性能进行压力测试来模拟现实世界的网站框架使用情况。我们发现它很好地反映了现实世界的加载和交互行为，并且我们一再发现，速度计的改进反映在我们的现实世界指标中。

使用Sparkplug，速度计分数提高了5-10%，具体取决于我们正在查看的机器人。

![Median improvement in Speedometer score with Sparkplug, across several performance bots. Error bars indicate inter-quartile range.](/\_img/sparkplug/benchmark-speedometer.svg)

# 浏览基准测试

车速表是一个很好的基准，但它只讲述了故事的一部分。我们还有一组“浏览基准测试”，它们是一组真实网站的记录，我们可以重播这些网站，编写一些交互脚本，并更真实地了解我们的各种指标在现实世界中的行为。

在这些基准测试中，我们选择查看我们的“V8 主线程时间”指标，该指标衡量 V8 在主线程上花费的总时间（包括编译和执行）（即不包括流式解析或后台优化编译）。这是了解Sparkplug在排除其他基准噪声源的同时为自己支付成本的最佳方式。

结果是多种多样的，并且非常依赖于机器和网站，但总的来说，它们看起来很棒：我们看到大约5-15%的改进。

：：： 图 V8 主线程时间在我们的浏览基准测试中位数改进，重复 10 次。误差线表示四分位数范围。
![Result for linux-perf bot](/\_img/sparkplug/benchmark-browsing-linux-perf.svg) ![Result for win-10-perf bot](/\_img/sparkplug/benchmark-browsing-win-10-perf.svg) ![Result for benchmark-browsing-mac-10\_13\_laptop_high_end-perf bot](/\_img/sparkplug/benchmark-browsing-mac-10\_13\_laptop_high_end-perf.svg) ![Result for mac-10\_12\_laptop_low_end-perf bot](/\_img/sparkplug/benchmark-browsing-mac-10\_12\_laptop_low_end-perf.svg) ![Result for mac-m1\_mini\_2020 bot](/\_img/sparkplug/benchmark-browsing-mac-m1\_mini\_2020-perf.svg)
:::

总而言之：V8 有一个新的超快非优化编译器，它将 V8 在实际基准测试上的性能提高了 5-15%。它已经在 V8 v9.1 中可用`--sparkplug`标记，我们将在Chrome 91中推出它。
