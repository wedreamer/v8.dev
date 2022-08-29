***

标题： “驯服V8中的架构复杂性 - CodeStubAssembler”
作者： '[丹尼尔·克利福德](https://twitter.com/expatdanno)， CodeStubAssembler assembler'
日期： 2017-11-16 13：33：37
标签：

*   内部
    描述： 'V8 在汇编代码之上有自己的抽象：CodeStubAssembler。CSA 允许 V8 在低级别快速可靠地优化 JS 功能，同时支持多个平台。
    推文：“931184976481177600”

***

在这篇文章中，我们想介绍CodeStubAssembler（CSA），这是V8中的一个组件，在实现一些目标方面非常有用。[大](/blog/optimizing-proxies) [性能](https://twitter.com/v8js/status/918119002437750784) [胜利](https://twitter.com../_gsathya/status/900188695721984000)在最近几个 V8 版本中。CSA还显著提高了V8团队在低级快速优化JavaScript功能的能力，提高了团队的开发速度。

## V8 中内置和手写组件的简要历史

要了解 CSA 在 V8 中的角色，重要的是要了解一些导致其发展的背景和历史。

V8 使用多种技术的组合从 JavaScript 中挤出性能。对于运行时间较长的 JavaScript 代码，V8 的[涡轮风扇](/docs/turbofan)优化编译器在加速整个ES2015 +功能范围以获得最佳性能方面做得很好。但是，V8 还需要有效地执行短运行 JavaScript，以获得良好的基准性能。对于所谓的**内置函数**上可用于所有 JavaScript 程序的预定义对象，如[ECMAScript 规范](https://tc39.es/ecma262/).

从历史上看，这些内置函数中的许多都是[自托管](https://en.wikipedia.org/wiki/Self-hosting)也就是说，它们是由 V8 开发人员用 JavaScript 编写的，尽管是一种特殊的 V8 内部方言。为了实现良好的性能，这些自托管内置组件依赖于 V8 用于优化用户提供的 JavaScript 的相同机制。与用户提供的代码一样，自承载内置组件需要一个预热阶段，在该阶段收集类型反馈，并且需要由优化编译器进行编译。

尽管此技术在某些情况下提供了良好的内置性能，但可以做得更好。上预定义函数的确切语义`Array.prototype`是[细节精致](https://tc39.es/ecma262/#sec-properties-of-the-array-prototype-object)在规范中。对于重要和常见的特殊情况，V8 的实现者通过了解规范，事先确切地知道这些内置函数应该如何工作，并且他们利用这些知识预先精心制作自定义的手动调整版本。这些*优化的内置*处理常见情况时无需预热或无需调用优化编译器，因为通过构造，基线性能在首次调用时已经是最佳的。

为了从手写的内置JavaScript函数（以及其他快速路径V8代码中挤出最佳性能，这些代码也被称为内置代码），V8开发人员传统上用汇编语言编写优化的内置。通过使用汇编，手写的内置函数特别快，除其他外，避免通过蹦床对V8的C++代码进行昂贵的调用，并利用V8基于自定义寄存器的。[阿比](https://en.wikipedia.org/wiki/Application_binary_interface)它在内部用于调用 JavaScript 函数。

由于手写汇编的优点，V8多年来为内置人员积累了数以万计的手写汇编代码......*每个平台*.所有这些手写程序集内置功能对于提高性能都非常有用，但是新的语言功能总是在标准化，并且维护和扩展此手写程序集既费力又容易出错。

## 输入 CodeStubAssembler

V8 开发人员多年来一直纠结于两难境地：是否有可能创建具有手写组装优势的内置组件，同时又不易碎且难以维护？

随着TurboFan的出现，这个问题的答案终于是“是的”。TurboFan的后端使用跨平台[中间表示](https://en.wikipedia.org/wiki/Intermediate_representation)（IR） 用于低水平机器操作。这种低级机器IR被输入到指令选择器，寄存器分配器，指令调度器和代码生成器中，可在所有平台上生成非常好的代码。后端还了解 V8 手写程序集内置中使用的许多技巧，例如如何使用和调用基于自定义寄存器的 ABI，如何支持机器级尾部调用，以及如何在叶函数中省略堆栈帧的构造。这些知识使得 TurboFan 后端特别适合生成与 V8 其余部分良好集成的快速代码。

这种功能组合使得手写组件内置的强大且可维护的替代方案首次成为可能。该团队构建了一个新的V8组件- 称为CodeStubAssembler或CSA - 它定义了一种基于TurboFan后端构建的可移植汇编语言。CSA添加了一个API来直接生成TurboFan机器级IR，而无需编写和解析JavaScript或应用TurboFan的JavaScript特定优化。虽然这种快速的代码生成路径只有 V8 开发人员才能用于在内部加速 V8 引擎，但这种以跨平台方式生成优化汇编代码的有效路径直接有利于所有开发人员在使用 CSA 构建的内置组件中的 JavaScript 代码，包括 V8 解释器的性能关键字节码处理程序，[点火](/docs/ignition).

![The CSA and JavaScript compilation pipelines](../_img/csa/csa.svg)

CSA 接口包括非常低级的操作，对于任何曾经编写过汇编代码的人来说都很熟悉。例如，它包括“从给定地址加载此对象指针”和“将这两个 32 位数字相乘”等功能。CSA 在 IR 级别具有类型验证，可在编译时而不是运行时捕获许多正确性错误。例如，它可以确保 V8 开发人员不会意外地使用从内存加载的对象指针作为 32 位乘法的输入。这种类型验证对于手写的程序集存根根本无法实现。

## CSA 试驾

为了更好地了解 CSA 提供的内容，让我们看一个简单示例。我们将向 V8 添加一个新的内部内置，如果对象是 String，则从对象返回字符串长度。如果输入对象不是字符串，则内置将返回`undefined`.

首先，我们在`BUILTIN_LIST_BASE`V8 中的宏[`builtin-definitions.h`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-definitions.h)文件，该文件声明名为`GetStringLength`并指定它具有一个用常量标识的输入参数`kInputObject`:

```cpp
TFS(GetStringLength, kInputObject)
```

这`TFS`宏将内置内容声明为**T**乌尔博**F**使用标准代码的内置**S**tub链接，这仅仅意味着它使用CSA来生成其代码，并期望通过寄存器传递参数。

然后，我们可以定义内置内容[`builtins-string-gen.cc`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-string-gen.cc):

```cpp
TF_BUILTIN(GetStringLength, CodeStubAssembler) {
  Label not_string(this);

  // Fetch the incoming object using the constant we defined for
  // the first parameter.
  Node* const maybe_string = Parameter(Descriptor::kInputObject);

  // Check to see if input is a Smi (a special representation
  // of small numbers). This needs to be done before the IsString
  // check below, since IsString assumes its argument is an
  // object pointer and not a Smi. If the argument is indeed a
  // Smi, jump to the label |not_string|.
  GotoIf(TaggedIsSmi(maybe_string), &not_string);

  // Check to see if the input object is a string. If not, jump to
  // the label |not_string|.
  GotoIfNot(IsString(maybe_string), &not_string);

  // Load the length of the string (having ended up in this code
  // path because we verified it was string above) and return it
  // using a CSA "macro" LoadStringLength.
  Return(LoadStringLength(maybe_string));

  // Define the location of label that is the target of the failed
  // IsString check above.
  BIND(&not_string);

  // Input object isn't a string. Return the JavaScript undefined
  // constant.
  Return(UndefinedConstant());
}
```

请注意，在上面的示例中，使用了两种类型的指令。有*原始*CSA 指令，可直接转换为一个或两个装配指令，如`GotoIf`和`Return`.有一组固定的预定义 CSA 原始指令，大致对应于您在 V8 支持的芯片架构之一上找到的最常用的汇编指令。示例中的其他说明是*宏观*说明，如`LoadStringLength`,`TaggedIsSmi`和`IsString`，这是以内联方式输出一个或多个基元指令或宏指令的便利函数。宏指令用于封装常用的 V8 实现习语，以便于重用。它们可以是任意长的，V8 开发人员可以在需要时轻松定义新的宏指令。

在编译了包含上述更改的 V8 之后，我们可以运行`mksnapshot`，该工具编译内置内容以为 V8 的快照做好准备，并具有`--print-code`命令行选项。此选项为每个内置内容打印生成的程序集代码。如果我们`grep`为`GetStringLength`在输出中，我们在x64上得到以下结果（代码输出被清理了一下，使其更具可读性）：

```asm
  test al,0x1
  jz not_string
  movq rbx,[rax-0x1]
  cmpb [rbx+0xb],0x80
  jnc not_string
  movq rax,[rax+0xf]
  retl
not_string:
  movq rax,[r13-0x60]
  retl
```

在 32 位 ARM 平台上，以下代码由`mksnapshot`:

```asm
  tst r0, #1
  beq +28 -> not_string
  ldr r1, [r0, #-1]
  ldrb r1, [r1, #+7]
  cmp r1, #128
  bge +12 -> not_string
  ldr r0, [r0, #+7]
  bx lr
not_string:
  ldr r0, [r10, #+16]
  bx lr
```

尽管我们的新内置使用非标准（至少非C++）调用约定，但也可以为其编写测试用例。以下代码可以添加到[`test-run-stubs.cc`](https://cs.chromium.org/chromium/src/v8/test/cctest/compiler/test-run-stubs.cc)在所有平台上测试内置功能：

```cpp
TEST(GetStringLength) {
  HandleAndZoneScope scope;
  Isolate* isolate = scope.main_isolate();
  Heap* heap = isolate->heap();
  Zone* zone = scope.main_zone();

  // Test the case where input is a string
  StubTester tester(isolate, zone, Builtins::kGetStringLength);
  Handle<String> input_string(
      isolate->factory()->
        NewStringFromAsciiChecked("Oktoberfest"));
  Handle<Object> result1 = tester.Call(input_string);
  CHECK_EQ(11, Handle<Smi>::cast(result1)->value());

  // Test the case where input is not a string (e.g. undefined)
  Handle<Object> result2 =
      tester.Call(factory->undefined_value());
  CHECK(result2->IsUndefined(isolate));
}
```

有关将 CSA 用于不同类型内置组件的更多详细信息以及更多示例，请参阅[这个维基页面](/docs/csa-builtins).

## V8 开发人员速度倍增器

CSA 不仅仅是一种面向多个平台的通用汇编语言。与过去为每个架构手动编写代码相比，在实现新功能时，它可以更快地实现周转。它通过提供手写汇编的所有好处来做到这一点，同时保护开发人员免受其最危险的陷阱的影响：

*   借助 CSA，开发人员可以使用一组跨平台的低级基元编写内置代码，这些基元直接转换为汇编指令。CSA 的指令选择器确保此代码在 V8 面向的所有平台上都是最佳的，而无需 V8 开发人员成为每个平台汇编语言的专家。
*   CSA 的接口具有可选类型，以确保由低级生成的程序集操作的值属于代码作者期望的类型。
*   汇编指令之间的寄存器分配由 CSA 自动完成，而不是手动显式完成，包括构建堆栈框架，如果内置使用比可用寄存器多或进行调用，则将值溢出到堆栈。这消除了一整类微妙的，难以发现的bug，这些错误困扰着手写汇编内置。通过使生成的代码不那么脆弱，CSA 大大减少了编写正确的低级内置文件所需的时间。
*   CSA 了解 ABI 调用约定（包括标准C++和基于 V8 寄存器的内部调用约定），从而可以在 CSA 生成的代码与 V8 的其他部分之间轻松进行互操作。
*   由于 CSA 代码C++，因此很容易将常见的代码生成模式封装在宏中，这些宏可以在许多内置函数中轻松重用。
*   由于 V8 使用 CSA 为 Ignition 生成字节码处理程序，因此很容易将基于 CSA 的内置函数的功能直接内联到处理程序中，以提高解释器的性能。
*   V8 的测试框架支持从 C++ 测试 CSA 功能和 CSA 生成的内置组件，而无需编写程序集适配器。

总而言之，CSA 一直是 V8 开发的游戏规则改变者。它显著提高了团队优化 V8 的能力。这意味着我们能够为 V8 的嵌入器更快地优化更多的 JavaScript 语言。
