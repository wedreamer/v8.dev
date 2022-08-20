***

标题： '更快的 JavaScript 调用'
作者： '[维克多·戈麦斯](https://twitter.com/VictorBFG)，框架碎纸机'
化身：

*   “胜利者戈麦斯”
    日期： 2021-02-15
    标签：
*   内部
    描述： “通过删除参数适配器框架来加快 JavaScript 调用速度”
    推文：“1361337569057865735”

***

JavaScript允许调用参数数与预期参数数不同的函数，即可以传递比声明的形式参数更少或更多的参数。前一种情况称为“应用不足”，后者称为“过度应用”。

在应用程序不足的情况下，将为其余参数分配未定义的值。在过度应用的情况下，可以使用 rest 参数和`arguments`属性，或者它们只是多余的，它们可以被忽略。如今，许多Web/Node.js框架都使用此JS功能来接受可选参数并创建更灵活的API。

直到最近，V8才有一个特殊的机制来处理参数大小不匹配：参数适配器框架。不幸的是，参数适应是以性能为代价的，但在现代前端和中间件框架中通常需要。事实证明，通过一个聪明的技巧，我们可以删除这个额外的框架，简化V8代码库，并摆脱几乎所有的开销。

我们可以通过微基准计算删除参数适配器框架的性能影响。

```js
console.time();
function f(x, y, z) {}
for (let i = 0; i <  N; i++) {
  f(1, 2, 3, 4, 5);
}
console.timeEnd();
```

![Performance impact of removing the arguments adaptor frame, as measured through a micro-benchmark.](/\_img/v8-release-89/perf.svg)

该图显示，在 上运行时不再有开销[无 JIT 模式](https://v8.dev/blog/jitless)（点火）性能提高11.2%。使用时[涡轮风扇](https://v8.dev/docs/turbofan)，我们获得了高达40%的加速。

这个微模板是自然设计的，以最大化参数适配器框架的影响。然而，我们看到许多基准有了相当大的改进，例如[我们的内部 JSTests/Array 基准测试](https://chromium.googlesource.com/v8/v8/+/b7aa85fe00c521a704ca83cc8789354e86482a60/test/js-perf-test/JSTests.json)（7%）和[辛烷值2](https://github.com/chromium/octane)（理查兹为4.6%，厄利博耶为6.1%）。

## TL;DR： 反转参数

该项目的全部意义在于删除参数适配器框架，该框架在堆栈中访问其参数时为被调用方提供了一致的接口。为此，我们需要反转堆栈中的参数，并在包含实际参数计数的被调用方帧中添加一个新槽。下图显示了更改前后典型帧的示例。

![A typical JavaScript stack frame before and after removing the arguments adaptor frame.](/\_img/adaptor-frame/frame-diff.svg)

## 使 JavaScript 调用速度更快

为了理解我们为加快调用速度所做的工作，让我们看看 V8 如何执行调用以及参数适配器框架的工作原理。

当我们在 JS 中调用函数调用时，V8 中会发生什么？让我们假设以下JS脚本：

```js
function add42(x) {
  return x + 42;
}
add42(3);
```

![Flow of execution inside V8 during a function call.](/\_img/adaptor-frame/flow.svg)

## 点火

V8 是一个多层 VM。它的第一层称为[点火](https://v8.dev/docs/ignition)，它是一个带有累加器寄存器的字节码堆栈机器。V8 首先将代码编译为[点火字节码](https://medium.com/dailyjs/understanding-v8s-bytecode-317d46c94775).上述调用编译为以下内容：

    0d              LdaUndefined              ;; Load undefined into the accumulator
    26 f9           Star r2                   ;; Store it in register r2
    13 01 00        LdaGlobal [1]             ;; Load global pointed by const 1 (add42)
    26 fa           Star r1                   ;; Store it in register r1
    0c 03           LdaSmi [3]                ;; Load small integer 3 into the accumulator
    26 f8           Star r3                   ;; Store it in register r3
    5f fa f9 02     CallNoFeedback r1, r2-r3  ;; Invoke call

调用的第一个参数通常称为接收方。接收器是`this`对象在 JSFunction 中，并且每个 JS 函数调用都必须有一个。的字节码处理程序`CallNoFeedback`需要调用对象`r1`使用寄存器列表中的参数`r2-r3`.

在我们深入研究字节码处理程序之前，请注意寄存器在字节码中的编码方式。它们是负单字节整数：`r1`编码为`fa`,`r2`如`f9`和`r3`如`f8`.我们可以将任何寄存器 ri 称为`fb - i`，实际上，正如我们将看到的，正确的编码是`- 2 - kFixedFrameHeaderSize - i`.寄存器列表使用第一个寄存器和列表的大小进行编码，因此`r2-r3`是`f9 02`.

Ignition中有许多字节码调用处理程序。您可以看到它们的列表[这里](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/interpreter/bytecodes.h;drc=3965dcd5cb1141c90f32706ac7c965dc5c1c55b3;l=184).它们彼此略有不同。有一些字节码针对具有以下`undefined`接收器，用于属性调用、具有固定数量参数的调用或泛型调用。在这里我们分析`CallNoFeedback`这是一个通用调用，我们不会从执行中积累反馈。

此字节码的处理程序非常简单。它写在[`CodeStubAssembler`](https://v8.dev/docs/csa-builtins)，您可以查看[这里](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/interpreter/interpreter-generator.cc;drc=6cdb24a4ce9d4151035c1f133833137d2e2881d1;l=1467).从本质上讲，它尾随到依赖于架构的内置[`InterpreterPushArgsThenCall`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/builtins/x64/builtins-x64.cc;drc=8665f09771c6b8220d6020fe9b1ad60a4b0b6591;l=1277).

内置功能实质上将返回地址弹出到临时寄存器，推送所有参数（包括接收器）并推回返回地址。在这一点上，我们不知道被调用方是否是可调用对象，也不知道被调用方期望多少个参数，即其形式参数计数。

![State of the frame after the execution of InterpreterPushArgsThenCall built-in.](/\_img/adaptor-frame/normal-push.svg)

最终执行尾部调用到内置[`Call`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/builtins/x64/builtins-x64.cc;drc=8665f09771c6b8220d6020fe9b1ad60a4b0b6591;l=2256).在那里，它检查目标是否是正确的函数，构造函数或任何可调用的对象。它还读取`shared function info`结构以获取其正式参数计数。

如果被调用方是函数对象，则尾部调用到内置[`CallFunction`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/builtins/x64/builtins-x64.cc;drc=8665f09771c6b8220d6020fe9b1ad60a4b0b6591;l=2038)，其中发生了一堆检查，包括如果我们有一个`undefined`对象作为接收方。如果我们有一个`undefined`或`null`对象作为接收者，我们应该修补它以引用全局代理对象，根据[ECMA 规范](https://262.ecma-international.org/11.0/#sec-ordinarycallbindthis).

然后执行尾部调用到内置[`InvokeFunctionCode`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/codegen/x64/macro-assembler-x64.cc;drc=a723767935dec385818d1134ea729a4c3a3ddcfb;l=2781)，这将在没有参数不匹配的情况下调用字段所指向的任何内容`Code`在被叫方对象中。这可以是优化的功能，也可以是内置的[`InterpreterEntryTrampoline`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/builtins/x64/builtins-x64.cc;drc=8665f09771c6b8220d6020fe9b1ad60a4b0b6591;l=1037).

如果我们假设我们正在调用一个尚未优化的函数，Ignition蹦床将设置一个`IntepreterFrame`.您可以看到 V8 中帧类型的简要摘要[这里](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/execution/frame-constants.h;drc=574ac5d62686c3de8d782dc798337ce1355dc066;l=14).

在不过多地介绍接下来会发生什么的情况下，我们可以看到被叫方执行期间解释器帧的快照。

![The InterpreterFrame for the call add42(3).](/\_img/adaptor-frame/normal-frame.svg)

我们看到帧中有固定数量的槽：返回地址、上一帧指针、上下文、我们正在执行的当前函数对象、此函数的字节码数组以及我们正在执行的当前字节码的偏移量。最后，我们有一个专用于此函数的寄存器列表（您可以将它们视为函数局部变量）。这`add42`函数实际上没有任何寄存器，但调用方具有具有3个寄存器的类似帧。

正如预期的那样，add42是一个简单的函数：

    25 02             Ldar a0          ;; Load the first argument to the accumulator
    40 2a 00          AddSmi [42]      ;; Add 42 to it
    ab                Return           ;; Return the accumulator

请注意我们如何在`Ldar` *（负载累加器寄存器）*字节码：参数`1`(`a0`） 用数字编码`02`.事实上，任何参数的编码都很简单`[ai] = 2 + parameter_count - i - 1`和接收器`[this] = 2 + parameter_count`，或在此示例中`[this] = 3`.此处的参数计数不包括接收器。

我们现在能够理解为什么我们以这种方式对寄存器和参数进行编码。它们只是表示与帧指针的偏移量。然后，我们可以以相同的方式处理参数/寄存器负载和存储。帧指针中最后一个参数的偏移量为`2`（上一帧指针和返回地址）。这解释了`2`在编码中。解释器框架的固定部分是`6`插槽 （`4`从帧指针），因此寄存器零位于偏移处`-5`，即`fb`注册`1`在`fa`.聪明，对吧？

但是请注意，为了能够访问参数，函数必须知道堆栈中有多少个参数！索引`2`指向最后一个参数，无论有多少个参数！

的字节码处理程序`Return`将通过调用内置`LeaveInterpreterFrame`.这个内置本质上是读取函数对象以从帧中获取参数计数，弹出当前帧，恢复帧指针，将返回地址保存在暂存寄存器中，根据参数计数弹出参数并跳转到暂存器中的地址。

所有这些流程都很棒！但是，当我们调用参数少于或多于其参数计数的函数时，会发生什么情况呢？聪明的参数/寄存器访问将失败，我们如何在调用结束时清理参数？

## 参数适配器框架

现在让我们调用`add42`参数越来越少：

```js
add42();
add42(1, 2, 3);
```

我们之间的JS开发人员会知道，在第一种情况下，`x`将被分配`undefined`该函数将返回`undefined + 42 = NaN`.在第二种情况下，`x`将被分配`1`该函数将返回`43`，其余参数将被忽略。请注意，调用方不知道是否会发生这种情况。即使调用方检查参数计数，被调用方也可以使用 rest 参数或参数对象来访问所有其他参数。实际上，参数对象甚至可以在外部访问`add42`在草率模式下。

如果我们遵循与以前相同的步骤，我们将首先调用内置`InterpreterPushArgsThenCall`.它会将参数推送到堆栈，如下所示：

![State of the frames after the execution of InterpreterPushArgsThenCall built-in.](/\_img/adaptor-frame/adaptor-push.svg)

继续与之前相同的过程，我们检查被调用方是否是函数对象，获取其参数计数并将接收方修补到全局代理。最终我们到达`InvokeFunctionCode`.

在这里，而不是跳到`Code`在被叫方对象中。我们检查参数大小和参数计数之间是否存在不匹配，然后跳转到`ArgumentsAdaptorTrampoline`.

在这个内置的框架中，我们构建了一个额外的框架，臭名昭着的论点适配器框架。我不会解释内置内部发生了什么，而是在内置调用被调用方之前向您呈现框架的状态。`Code`.请注意，这是一个适当的`x64 call`（不是`jmp`），并且在被调用方执行后，我们将返回到`ArgumentsAdaptorTrampoline`.这与`InvokeFunctionCode`那个尾声。

![Stack frames with arguments adaptation.](/\_img/adaptor-frame/adaptor-frames.svg)

您可以看到，我们创建了另一个帧，该帧复制了所有必要的参数，以便将参数的参数精确地放在被调用方帧的顶部。它创建被调用方函数的接口，以便后者不需要知道参数的数量。被调用方将始终能够以与以前相同的计算方式访问其参数，即`[ai] = 2 + parameter_count - i - 1`.

V8 具有特殊的内置功能，每当适配器帧需要通过 rest 参数或参数对象访问其余参数时，它都能理解适配器帧。他们总是需要检查被叫方框架顶部的适配器框架类型，然后采取相应的行动。

如您所见，我们解决了参数/寄存器访问问题，但我们创造了很多复杂性。每个需要访问所有参数的内置都需要理解并检查适配器框架的存在。不仅如此，我们需要小心不要访问陈旧和旧数据。请考虑以下更改`add42`:

```js
function add42(x) {
  x += 42;
  return x;
}
```

字节码数组现在是：

    25 02             Ldar a0       ;; Load the first argument to the accumulator
    40 2a 00          AddSmi [42]   ;; Add 42 to it
    26 02             Star a0       ;; Store accumulator in the first argument slot
    ab                Return        ;; Return the accumulator

如您所见，我们现在修改`a0`.因此，在呼叫的情况下`add42(1, 2, 3)`参数适配器帧中的插槽将被修改，但调用方帧仍将包含该号码`1`.我们需要小心，参数对象正在访问修改后的值，而不是过时的值。

从函数返回很简单，尽管速度很慢。记住什么`LeaveInterpreterFrame`吗？它基本上弹出被调用方帧和参数计数号的参数。因此，当我们返回到参数适配器存根时，堆栈如下所示：

![State of the frames after the execution of the callee add42.](/\_img/adaptor-frame/adaptor-frames-cleanup.svg)

我们只需要弹出参数的数量，弹出适配器框架，根据实际参数计数弹出所有参数并返回调用方执行。

TL;DR：参数适配器机器不仅复杂，而且成本高昂。

## 删除参数适配器框架

我们能做得更好吗？我们可以移除适配器框架吗？事实证明，我们确实可以。

让我们回顾一下我们的要求：

1.  我们需要能够像以前一样无缝地访问参数和寄存器。访问它们时无法进行任何检查。那太贵了。
2.  我们需要能够从堆栈中构造 rest 参数和参数对象。
3.  我们需要能够在从调用返回时轻松清理未知数量的参数。
4.  而且，当然，我们希望在没有额外框架的情况下做到这一点！

如果我们想消除多余的帧，那么我们需要决定将参数放在何处：在被叫方帧中或在调用方帧中。

### 被叫方框架中的参数

假设我们将参数放在被调用方框架中。这实际上是一个好主意，因为每当我们弹出框架时，我们也会立即弹出所有论点！

参数需要位于保存的帧指针和帧末尾之间的某个位置。这意味着帧的大小不会被静态地知道。访问参数仍然很容易，它是从帧指针的简单偏移。但是访问寄存器现在要复杂得多，因为它根据参数的数量而变化。

堆栈指针始终指向最后一个寄存器，然后我们可以使用它来访问寄存器，而无需知道参数计数。这种方法可能实际上有效，但它有一个主要缺点。这将需要复制所有可以访问寄存器和参数的字节码。我们需要一个`LdaArgument`和`LdaRegister`而不是简单地`Ldar`.当然，我们也可以检查我们是否正在访问参数或寄存器（正偏移量或负偏移量），但这需要检查每个参数并访问寄存器。显然太贵了！

### 调用方框架中的参数

好。。。如果我们坚持使用调用方帧中的参数会怎样？

记住如何计算参数的偏移量`i`在框架中：`[ai] = 2 + parameter_count - i - 1`.如果我们有所有参数（不仅仅是参数），则偏移量将为`[ai] = 2 + argument_count - i - 1`.也就是说，对于每个参数访问，我们都需要加载实际的参数计数。

但是，如果我们颠倒论点会发生什么呢？现在，偏移量可以简单地计算为`[ai] = 2 + i`.我们不需要知道堆栈中有多少个参数，但是如果我们能保证我们总是在堆栈中至少有参数的参数计数，那么我们总是可以使用这个方案来计算偏移量。

换句话说，堆栈中推送的参数数将始终是参数数和形式参数计数之间的最大值，如果需要，将使用未定义的对象填充它。

这还有另一个好处！接收器始终位于任何JS函数的相同偏移量中，就在返回地址的正上方：`[this] = 2`.

这是我们要求编号的干净解决方案`1`和数字`4`.那么另外两个要求呢？我们如何构造 rest 参数和参数对象？返回调用方时如何清理堆栈中的参数？为此，我们只是缺少参数计数。我们需要把它保存在某个地方。这里的选择有点武断，只要很容易访问这些信息。两个基本选择是：在调用方帧中紧跟在接收方之后推送它，或者在固定标头部分中将其作为被调用方帧的一部分。我们实现了后者，因为它合并了解释器和优化帧的固定标头部分。

如果我们在 V8 v8.9 中运行我们的示例，我们将在以下位置看到以下堆栈：`InterpreterArgsThenPush`（请注意，参数现在被反转）：

![State of the frames after the execution of InterpreterPushArgsThenCall built-in.](/\_img/adaptor-frame/no-adaptor-push.svg)

所有执行都遵循类似的路径，直到我们到达 InvokeFunctionCode。在这里，我们在应用不足的情况下调整参数，根据需要推送尽可能多的未定义对象。请注意，在过度应用的情况下，我们不会更改任何内容。最后，我们将参数的数量传递给被调用方`Code`通过寄存器。在以下情况下`x64`，我们使用寄存器`rax`.

如果被叫方尚未优化，我们会达到`InterpreterEntryTrampoline`，这将生成以下堆栈帧。

![Stack frames without arguments adaptors.](/\_img/adaptor-frame/no-adaptor-frames.svg)

被调用方帧有一个额外的槽，其中包含可用于构造 rest 参数或参数对象的参数数，以及在返回给调用方之前清理堆栈中的参数。

为了返回，我们修改`LeaveInterpreterFrame`以读取堆栈中的参数计数，并弹出参数计数和正式参数计数之间的最大数字。

## 涡轮风扇

那么优化的代码呢？让我们稍微改变一下初始脚本，强制 V8 使用 TurboFan 编译它：

```js
function add42(x) { return x + 42; }
function callAdd42() { add42(3); }
%PrepareFunctionForOptimization(callAdd42);
callAdd42();
%OptimizeFunctionOnNextCall(callAdd42);
callAdd42();
```

在这里，我们使用 V8 内部函数来强制 V8 优化调用，否则 V8 只会在小函数变热时才优化我们的小函数（经常使用）。我们在优化之前调用它一次，以收集一些可用于指导编译的类型信息。阅读更多 关于 涡轮风扇[这里](https://v8.dev/docs/turbofan).

我在这里只向您展示生成的代码中与我们相关的部分。

```nasm
movq rdi,0x1a8e082126ad    ;; Load the function object <JSFunction add42>
push 0x6                   ;; Push SMI 3 as argument
movq rcx,0x1a8e082030d1    ;; <JSGlobal Object>
push rcx                   ;; Push receiver (the global proxy object)
movl rax,0x1               ;; Save the arguments count in rax
movl rcx,[rdi+0x17]        ;; Load function object {Code} field in rcx
call rcx                   ;; Finally, call the code object!
```

虽然是用汇编程序编写的，但如果你按照我的评论，这个代码片段应该不难阅读。从本质上讲，在编译调用时，TF 需要完成在`InterpreterPushArgsThenCall`,`Call`,`CallFunction`和`InvokeFunctionCall`内置。希望它有更多的静态信息来做到这一点，并发出更少的计算机指令。

### 涡轮风扇与参数适配器框架

现在，让我们看看参数数量和参数计数不匹配的情况。考虑呼叫`add42(1, 2, 3)`.这被编译为：

```nasm
movq rdi,0x4250820fff1    ;; Load the function object <JSFunction add42>
;; Push receiver and arguments SMIs 1, 2 and 3
movq rcx,0x42508080dd5    ;; <JSGlobal Object>
push rcx
push 0x2
push 0x4
push 0x6
movl rax,0x3              ;; Save the arguments count in rax
movl rbx,0x1              ;; Save the formal parameters count in rbx
movq r10,0x564ed7fdf840   ;; <ArgumentsAdaptorTrampoline>
call r10                  ;; Call the ArgumentsAdaptorTrampoline
```

如您所见，不难为 TF 添加对参数和参数计数不匹配的支持。只需将参数称为适配器蹦床！

然而，这很昂贵。对于每个优化的调用，我们现在需要输入参数适配器蹦床，并像在未优化代码中一样按摩框架。这就解释了为什么在优化代码中删除适配器框架的性能增益比在Ignition上大得多。

但是，生成的代码非常简单。从中返回非常容易（尾声）：

```nasm
movq rsp,rbp   ;; Clean callee frame
pop rbp
ret 0x8        ;; Pops a single argument (the receiver)
```

我们弹出我们的框架，并根据参数计数发出返回指令。如果我们的参数数量和参数计数不匹配，适配器框架蹦床将处理它。

### 涡轮风扇不带参数适配器框架

生成的代码本质上与具有匹配数量的参数的调用中的代码相同。考虑呼叫`add42(1, 2, 3)`.这将生成：

```nasm
movq rdi,0x35ac082126ad    ;; Load the function object <JSFunction add42>
;; Push receiver and arguments 1, 2 and 3 (reversed)
push 0x6
push 0x4
push 0x2
movq rcx,0x35ac082030d1    ;; <JSGlobal Object>
push rcx
movl rax,0x3               ;; Save the arguments count in rax
movl rcx,[rdi+0x17]        ;; Load function object {Code} field in rcx
call rcx                   ;; Finally, call the code object!
```

函数的尾声呢？我们不再回到适应蹦床的论点，所以尾声确实比以前复杂了一点。

```nasm
movq rcx,[rbp-0x18]        ;; Load the argument count (from callee frame) to rcx
movq rsp,rbp               ;; Pop out callee frame
pop rbp
cmpq rcx,0x0               ;; Compare arguments count with formal parameter count
jg 0x35ac000840c6  <+0x86>
;; If arguments count is smaller (or equal) than the formal parameter count:
ret 0x8                    ;; Return as usual (parameter count is statically known)
;; If we have more arguments in the stack than formal parameters:
pop r10                    ;; Save the return address
leaq rsp,[rsp+rcx*8+0x8]   ;; Pop all arguments according to rcx
push r10                   ;; Recover the return address
retl
```

# 结论

参数适配器框架是一种临时解决方案，用于处理参数和形式参数数量不匹配的调用。这是一个简单的解决方案，但它带来了高性能成本，并增加了代码库的复杂性。如今，许多 Web 框架使用此功能来创建更灵活的 API，从而加剧了性能成本。反转堆栈中参数的简单想法可以显着降低实现复杂性，并消除此类调用的几乎全部开销。
