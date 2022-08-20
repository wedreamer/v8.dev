***

## 标题： '内置V8扭矩'&#xA;描述： “本文档旨在介绍Torque内置的编写，并针对V8开发人员。

本文档旨在介绍Torque内置的编写，面向V8开发人员。Torque取代CodeStubAssembler成为实现新内置组件的推荐方法。看[CodeStubAssembler builtins](/docs/csa-builtins)对于本指南的 CSA 版本。

## 内置

在 V8 中，内置组件可以看作是 VM 在运行时可执行的代码块。一个常见的用例是实现内置对象的功能（例如`RegExp`或`Promise`），但内置功能也可用于提供其他内部功能（例如，作为IC系统的一部分）。

V8 的内置功能可以使用许多不同的方法实现（每种方法都有不同的权衡）：

*   **依赖于平台的汇编语言**：可以非常高效，但需要手动端口到所有平台并且难以维护。
*   **C++**：风格与运行时函数非常相似，可以访问 V8 强大的运行时功能，但通常不适合对性能敏感的区域。
*   **JavaScript**：简洁易读的代码，访问快速的内部函数，但频繁使用缓慢的运行时调用，通过类型污染获得不可预测的性能，以及围绕（复杂且不明显）JS语义的微妙问题。Javascript内置的已经不推荐使用，不应该再添加。
*   **CodeStubAssembler**：提供非常接近汇编语言的高效低级功能，同时保持独立于平台并保持可读性。
*   **[V8 扭矩](/docs/torque)**：是一种特定于 V8 的域特定语言，已翻译为 CodeStubAssembler。因此，它扩展了CodeStubAssembler，并提供静态类型以及可读和富有表现力的语法。

剩下的文档重点介绍后者，并给出了一个简短的教程，用于开发一个暴露在JavaScript中的简单Torque内置。有关扭矩的更完整信息，请参阅[V8 扭矩用户手册](/docs/torque).

## 编写内置扭矩

在本节中，我们将编写一个简单的 CSA 内置，该内置函数采用单个参数，并返回它是否表示数字`42`.内置组件通过安装在`Math`对象（因为我们可以）。

此示例演示：

*   使用JavaScript链接创建一个内置的扭矩，可以像JS函数一样调用。
*   使用扭矩实现简单的逻辑：类型区分，Smi和堆数处理，条件。
*   在`Math`对象。

如果您想在本地进行操作，以下代码基于修订版[589af9f2](https://chromium.googlesource.com/v8/v8/+/589af9f257166f66774b4fb3008cd09f192c2614).

## 定义`MathIs42`

扭矩代码位于`src/builtins/*.tq`文件，大致按主题组织。由于我们将编写一个`Math`内置，我们将把我们的定义`src/builtins/math.tq`.由于此文件尚不存在，因此我们必须将其添加到[`torque_files`](https://cs.chromium.org/chromium/src/v8/BUILD.gn?l=914\&rcl=589af9f257166f66774b4fb3008cd09f192c2614)在[`BUILD.gn`](https://cs.chromium.org/chromium/src/v8/BUILD.gn).

```torque
namespace math {
  javascript builtin MathIs42(
      context: Context, receiver: Object, x: Object): Boolean {
    // At this point, x can be basically anything - a Smi, a HeapNumber,
    // undefined, or any other arbitrary JS object. ToNumber_Inline is defined
    // in CodeStubAssembler. It inlines a fast-path (if the argument is a number
    // already) and calls the ToNumber builtin otherwise.
    const number: Number = ToNumber_Inline(x);
    // A typeswitch allows us to switch on the dynamic type of a value. The type
    // system knows that a Number can only be a Smi or a HeapNumber, so this
    // switch is exhaustive.
    typeswitch (number) {
      case (smi: Smi): {
        // The result of smi == 42 is not a Javascript boolean, so we use a
        // conditional to create a Javascript boolean value.
        return smi == 42 ? True : False;
      }
      case (heapNumber: HeapNumber): {
        return Convert<float64>(heapNumber) == 42 ? True : False;
      }
    }
  }
}
```

我们将定义放在 Torque 命名空间中`math`.由于此命名空间以前不存在，因此我们必须将其添加到[`torque_namespaces`](https://cs.chromium.org/chromium/src/v8/BUILD.gn?l=933\&rcl=589af9f257166f66774b4fb3008cd09f192c2614)在[`BUILD.gn`](https://cs.chromium.org/chromium/src/v8/BUILD.gn).

## 附加`Math.is42`

内置对象，例如`Math`主要设置在[`src/bootstrapper.cc`](https://cs.chromium.org/chromium/src/v8/src/bootstrapper.cc?q=src/bootstrapper.cc+package:%5Echromium$\&l=1)（在`.js`文件）。附加我们的新内置很简单：

```cpp
// Existing code to set up Math, included here for clarity.
Handle<JSObject> math = factory->NewJSObject(cons, TENURED);
JSObject::AddProperty(global, name, math, DONT_ENUM);
// […snip…]
SimpleInstallFunction(isolate_, math, "is42", Builtins::kMathIs42, 1, true);
```

既然`is42`是附加的，它可以从JS调用：

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

内置也可以使用存根链接创建（而不是我们上面使用的JS链接）`MathIs42`).此类内置代码可用于将常用代码提取到可由多个调用方使用的单独代码对象中，而代码仅生成一次。让我们将处理堆数的代码提取到一个单独的内置中，称为`HeapNumberIs42`，并从`MathIs42`.

定义也很简单。与Javascript链接的内置的唯一区别是我们省略了关键字`javascript`并且没有接收器参数。

```torque
namespace math {
  builtin HeapNumberIs42(implicit context: Context)(heapNumber: HeapNumber):
      Boolean {
    return Convert<float64>(heapNumber) == 42 ? True : False;
  }

  javascript builtin MathIs42(implicit context: Context)(
      receiver: Object, x: Object): Boolean {
    const number: Number = ToNumber_Inline(x);
    typeswitch (number) {
      case (smi: Smi): {
        return smi == 42 ? True : False;
      }
      case (heapNumber: HeapNumber): {
        // Instead of handling heap numbers inline, we now call our new builtin.
        return HeapNumberIs42(heapNumber);
      }
    }
  }
}
```

你为什么要关心内在的东西呢？为什么不让代码保持内联（或提取到宏中以提高可读性）？

一个重要原因是代码空间：内置组件在编译时生成，并包含在 V8 快照中或嵌入到二进制文件中。提取大量常用代码来分离内置组件可以快速节省10到100 KB的空间。

## 测试内置的存根链接

尽管我们的新内置使用非标准（至少非C++）调用约定，但也可以为其编写测试用例。以下代码可以添加到[`test/cctest/compiler/test-run-stubs.cc`](https://cs.chromium.org/chromium/src/v8/test/cctest/compiler/test-run-stubs.cc)在所有平台上测试内置功能：

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
