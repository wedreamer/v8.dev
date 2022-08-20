***

## 标题： 'CodeStubAssembler builtins'&#xA;描述：“本文档旨在介绍编写 CodeStubAssembler 内置内容，面向 V8 开发人员。

本文档旨在介绍编写 CodeStubAssembler 内置内容，面向 V8 开发人员。

：：：备注
**注意：** [力矩](/docs/torque)取代 CodeStubAssembler 作为实现新内置组件的推荐方法。看[内置扭矩](/docs/torque-builtins)对于本指南的扭矩版本。
:::

## 内置

在 V8 中，内置组件可以看作是 VM 在运行时可执行的代码块。一个常见的用例是实现内置对象的功能（例如RegExp或Promise），但内置也可用于提供其他内部功能（例如作为IC系统的一部分）。

V8 的内置功能可以使用许多不同的方法实现（每种方法都有不同的权衡）：

*   **依赖于平台的汇编语言**：可以非常高效，但需要手动端口到所有平台并且难以维护。
*   **C++**：风格与运行时函数非常相似，可以访问 V8 强大的运行时功能，但通常不适合对性能敏感的区域。
*   **JavaScript**：简洁易读的代码，访问快速的内部函数，但频繁使用缓慢的运行时调用，通过类型污染获得不可预测的性能，以及围绕（复杂且不明显）JS语义的微妙问题。
*   **CodeStubAssembler**：提供非常接近汇编语言的高效低级功能，同时保持独立于平台并保持可读性。

剩下的文档重点介绍后者，并给出了一个简短的教程，用于开发一个简单的CodeStubAssembler（CSA）内置于JavaScript。

## CodeStubAssembler

V8 的 CodeStubAssembler 是一个自定义的、与平台无关的汇编程序，它提供低级基元作为程序集的精简抽象，但也提供了一个广泛的高级功能库。

```cpp
// Low-level:
// Loads the pointer-sized data at addr into value.
Node* addr = /* ... */;
Node* value = Load(MachineType::IntPtr(), addr);

// And high-level:
// Performs the JS operation ToString(object).
// ToString semantics are specified at https://tc39.es/ecma262/#sec-tostring.
Node* object = /* ... */;
Node* string = ToString(context, object);
```

CSA内置组件通过TurboFan编译管道的一部分运行（包括块调度和寄存器分配，但显然不是通过优化传递），然后发出最终的可执行代码。

## 编写内置的 CodeStubAssembler

在本节中，我们将编写一个简单的 CSA 内置，该内置函数采用单个参数，并返回它是否表示数字`42`.内置组件通过安装在`Math`对象（因为我们可以）。

此示例演示：

*   使用JavaScript链接创建内置的CSA，可以像JS函数一样调用。
*   使用 CSA 实现简单逻辑：Smi 和堆号处理、条件以及对 TFS 内置的调用。
*   使用 CSA 变量。
*   在`Math`对象。

如果您想在本地进行操作，以下代码基于修订版[7a8d20a7](https://chromium.googlesource.com/v8/v8/+/7a8d20a79f9d5ce6fe589477b09327f3e90bf0e0).

## 声明`MathIs42`

内置项声明在`BUILTIN_LIST_BASE`宏[`src/builtins/builtins-definitions.h`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-definitions.h?q=builtins-definitions.h+package:%5Echromium$\&l=1).使用 JS 链接和一个名为`X`:

```cpp
#define BUILTIN_LIST_BASE(CPP, API, TFJ, TFC, TFS, TFH, ASM, DBG)              \
  // […snip…]
  TFJ(MathIs42, 1, kX)                                                         \
  // […snip…]
```

请注意，`BUILTIN_LIST_BASE`采用几个不同的宏来表示不同的内置类型（有关更多详细信息，请参阅内联文档）。CSA 内置功能具体分为：

*   **断续器**：JavaScript 链接。
*   **断续器**：存根链接。
*   **断续器**：内置的存根链接需要自定义接口描述符（例如，如果参数未标记或需要在特定寄存器中传递）。
*   **断续器**：内置专用短截线连杆，用于 IC 处理程序。

## 定义`MathIs42`

内置定义位于`src/builtins/builtins-*-gen.cc`文件，大致按主题组织。由于我们将编写一个`Math`内置，我们将把我们的定义[`src/builtins/builtins-math-gen.cc`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-math-gen.cc?q=builtins-math-gen.cc+package:%5Echromium$\&l=1).

```cpp
// TF_BUILTIN is a convenience macro that creates a new subclass of the given
// assembler behind the scenes.
TF_BUILTIN(MathIs42, MathBuiltinsAssembler) {
  // Load the current function context (an implicit argument for every stub)
  // and the X argument. Note that we can refer to parameters by the names
  // defined in the builtin declaration.
  Node* const context = Parameter(Descriptor::kContext);
  Node* const x = Parameter(Descriptor::kX);

  // At this point, x can be basically anything - a Smi, a HeapNumber,
  // undefined, or any other arbitrary JS object. Let’s call the ToNumber
  // builtin to convert x to a number we can use.
  // CallBuiltin can be used to conveniently call any CSA builtin.
  Node* const number = CallBuiltin(Builtins::kToNumber, context, x);

  // Create a CSA variable to store the resulting value. The type of the
  // variable is kTagged since we will only be storing tagged pointers in it.
  VARIABLE(var_result, MachineRepresentation::kTagged);

  // We need to define a couple of labels which will be used as jump targets.
  Label if_issmi(this), if_isheapnumber(this), out(this);

  // ToNumber always returns a number. We need to distinguish between Smis
  // and heap numbers - here, we check whether number is a Smi and conditionally
  // jump to the corresponding labels.
  Branch(TaggedIsSmi(number), &if_issmi, &if_isheapnumber);

  // Binding a label begins generating code for it.
  BIND(&if_issmi);
  {
    // SelectBooleanConstant returns the JS true/false values depending on
    // whether the passed condition is true/false. The result is bound to our
    // var_result variable, and we then unconditionally jump to the out label.
    var_result.Bind(SelectBooleanConstant(SmiEqual(number, SmiConstant(42))));
    Goto(&out);
  }

  BIND(&if_isheapnumber);
  {
    // ToNumber can only return either a Smi or a heap number. Just to make sure
    // we add an assertion here that verifies number is actually a heap number.
    CSA_ASSERT(this, IsHeapNumber(number));
    // Heap numbers wrap a floating point value. We need to explicitly extract
    // this value, perform a floating point comparison, and again bind
    // var_result based on the outcome.
    Node* const value = LoadHeapNumberValue(number);
    Node* const is_42 = Float64Equal(value, Float64Constant(42));
    var_result.Bind(SelectBooleanConstant(is_42));
    Goto(&out);
  }

  BIND(&out);
  {
    Node* const result = var_result.value();
    CSA_ASSERT(this, IsBoolean(result));
    Return(result);
  }
}
```

## 附加`Math.Is42`

内置对象，例如`Math`主要设置在[`src/bootstrapper.cc`](https://cs.chromium.org/chromium/src/v8/src/bootstrapper.cc?q=src/bootstrapper.cc+package:%5Echromium$\&l=1)（在`.js`文件）。附加我们的新内置很简单：

```cpp
// Existing code to set up Math, included here for clarity.
Handle<JSObject> math = factory->NewJSObject(cons, TENURED);
JSObject::AddProperty(global, name, math, DONT_ENUM);
// […snip…]
SimpleInstallFunction(math, "is42", Builtins::kMathIs42, 1, true);
```

既然`Is42`是附加的，它可以从JS调用：

```bash
$ out/debug/d8
d8> Math.is42(42);
true
d8> Math.is42('42.0');
true
d8> Math.is42(true);
false
d8> Math.is42({ valueOf: () => 42 });
true
```

## 定义和调用具有存根链接的内置

CSA内置功能也可以使用存根链接创建（而不是我们上面使用的JS链接）`MathIs42`).此类内置代码可用于将常用代码提取到可由多个调用方使用的单独代码对象中，而代码仅生成一次。让我们将处理堆数的代码提取到一个单独的内置中，称为`MathIsHeapNumber42`，并从`MathIs42`.

定义和使用TFS存根很容易;声明再次放在[`src/builtins/builtins-definitions.h`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-definitions.h?q=builtins-definitions.h+package:%5Echromium$\&l=1):

```cpp
#define BUILTIN_LIST_BASE(CPP, API, TFJ, TFC, TFS, TFH, ASM, DBG)              \
  // […snip…]
  TFS(MathIsHeapNumber42, kX)                                                  \
  TFJ(MathIs42, 1, kX)                                                         \
  // […snip…]
```

请注意，目前，订单在`BUILTIN_LIST_BASE`确实很重要。因为`MathIs42`调用`MathIsHeapNumber42`，前者需要在后者之后列出（此要求应在某个时候取消）。

定义也很简单。在[`src/builtins/builtins-math-gen.cc`](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-math-gen.cc?q=builtins-math-gen.cc+package:%5Echromium$\&l=1):

```cpp
// Defining a TFS builtin works exactly the same way as TFJ builtins.
TF_BUILTIN(MathIsHeapNumber42, MathBuiltinsAssembler) {
  Node* const x = Parameter(Descriptor::kX);
  CSA_ASSERT(this, IsHeapNumber(x));
  Node* const value = LoadHeapNumberValue(x);
  Node* const is_42 = Float64Equal(value, Float64Constant(42));
  Return(SelectBooleanConstant(is_42));
}
```

最后，让我们调用我们的新内置`MathIs42`:

```cpp
TF_BUILTIN(MathIs42, MathBuiltinsAssembler) {
  // […snip…]
  BIND(&if_isheapnumber);
  {
    // Instead of handling heap numbers inline, we now call into our new TFS stub.
    var_result.Bind(CallBuiltin(Builtins::kMathIsHeapNumber42, context, number));
    Goto(&out);
  }
  // […snip…]
}
```

你为什么要关心 TFS 内置？为什么不将代码保持内联（或提取到帮助器方法中以提高可读性）？

一个重要原因是代码空间：内置组件在编译时生成并包含在 V8 快照中，因此在每个创建的隔离中无条件地占用（重要）空间。将大量常用代码提取到 TFS 内置组件可以快速节省 10 到 100 KB 的空间。

## 测试内置的存根链接

尽管我们的新内置使用非标准（至少非C++）调用约定，但也可以为其编写测试用例。以下代码可以添加到[`test/cctest/compiler/test-run-stubs.cc`](https://cs.chromium.org/chromium/src/v8/test/cctest/compiler/test-run-stubs.cc?l=1\&rcl=4cab16db27808cf66ab883e7904f1891f9fd0717)在所有平台上测试内置功能：

```cpp
TEST(MathIsHeapNumber42) {
  HandleAndZoneScope scope;
  Isolate* isolate = scope.main_isolate();
  Heap* heap = isolate->heap();
  Zone* zone = scope.main_zone();

  StubTester tester(isolate, zone, Builtins::kMathIs42);
  Handle<Object> result1 = tester.Call(Handle<Smi>(Smi::FromInt(0), isolate));
  CHECK(result1->BooleanValue());
}
```
