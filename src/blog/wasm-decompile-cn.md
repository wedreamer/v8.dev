***

标题： '那是什么`.wasm`?正在推出：`wasm-decompile`'
作者： 'Wouter van Oortmerssen （[@wvo](https://twitter.com/wvo))'
化身：

*   'wouter-van-oortmerssen'
    日期： 2020-04-27
    标签：
*   WebAssembly
*   工具
    描述： “WABT获得了一个新的反编译工具，可以更容易地阅读Wasm模块的内容。
    推文：“1254829913561014272”

***

我们有越来越多的编译器和其他工具来生成或操作`.wasm`文件，有时您可能希望查看内部。也许你是这种工具的开发人员，或者更直接地说，你是一个针对Wasm的程序员，并且出于性能或其他原因想知道生成的代码是什么样子的。

问题是，Wasm是相当低级的，就像实际的汇编代码一样。特别是，与JVM不同，所有数据结构都被编译为加载/存储操作，而不是方便命名的类和字段。像LLVM这样的编译器可以进行大量的转换，使生成的代码看起来与进入的代码完全不同。

## 拆卸或..反编译？

您可以使用以下工具：`wasm2wat`（部分[瓦布特](https://github.com/WebAssembly/wabt)工具包），以转换`.wasm`转换为 Wasm 的标准文本格式，`.wat`，这是一个非常忠实但不是特别可读的表示。

例如，一个简单的 C 函数，如点积：

```c
typedef struct { float x, y, z; } vec3;

float dot(const vec3 *a, const vec3 *b) {
    return a->x * b->x +
           a->y * b->y +
           a->z * b->z;
}
```

我们使用`clang dot.c -c -target wasm32 -O2`其次`wasm2wat -f dot.o`把它变成这个`.wat`:

```wasm
(func $dot (type 0) (param i32 i32) (result f32)
  (f32.add
    (f32.add
      (f32.mul
        (f32.load
          (local.get 0))
        (f32.load
          (local.get 1)))
      (f32.mul
        (f32.load offset=4
          (local.get 0))
        (f32.load offset=4
          (local.get 1))))
    (f32.mul
      (f32.load offset=8
        (local.get 0))
      (f32.load offset=8
        (local.get 1))))))
```

这是一小段代码，但由于许多原因已经不是很好阅读。除了缺乏基于表达式的语法和一般冗长之外，必须将数据结构理解为内存负载并不容易。现在想象一下，看着一个大程序的输出，事情很快就会变得难以理解。

而不是`wasm2wat`跑`wasm-decompile dot.o`，您将获得：

```c
function dot(a:{ a:float, b:float, c:float },
             b:{ a:float, b:float, c:float }):float {
  return a.a * b.a + a.b * b.b + a.c * b.c
}
```

这看起来要熟悉得多。除了模仿您可能熟悉的编程语言的基于表达式的语法之外，反编译器还会查看函数中的所有加载和存储，并尝试推断其结构。然后，它使用“内联”结构声明对用作指针的每个变量进行批注。它不会创建命名的结构声明，因为它不一定知道 3 个浮点数的哪些用法表示相同的概念。

## 反编译成什么？

`wasm-decompile`产生的输出试图看起来像一种“非常普通的编程语言”，同时仍然接近它所代表的Wasm。

它的#1目标是可读性：帮助引导读者理解`.wasm`尽可能易于遵循的代码。它的#2目标是仍然以1：1的方式表示Wasm，以免失去其作为拆卸器的实用性。显然，这两个目标并不总是统一的。

此输出不是实际的编程语言，目前无法将其编译回 Wasm。

### 装载和存储

如上所述，`wasm-decompile`查看特定指针上的所有加载和存储。如果它们形成一组连续的访问，它将输出这些“内联”结构声明之一。

如果不是所有的“字段”都被访问，它就无法确定这是一个结构，还是某种其他形式的不相关的内存访问。在这种情况下，它会回退到更简单的类型，例如`float_ptr`（如果类型相同），或者，在最坏的情况下，将输出数组访问，如`o[2]:int`，其中说：`o`指向`int`值，我们正在访问第三个值。

最后一种情况发生的频率比您想象的要高，因为 Wasm 局部变量的功能更像是寄存器而不是变量，因此优化的代码可能会为不相关的对象共享相同的指针。

反编译器试图在索引方面保持聪明，并检测诸如`(base + (index << 2))[0]:int`由常规 C 数组索引操作产生，例如`base[index]`哪里`base`指向 4 字节类型。这些在代码中很常见，因为Wasm在负载和存储上只有恒定的偏移量。`wasm-decompile`输出将它们转换回`base[index]:int`.

此外，它还知道绝对地址何时引用数据部分。

### 控制流

最熟悉的是Wasm的if-then构造，它转化为熟悉的`if (cond) { A } else { B }`语法，加上在Wasm中它实际上可以返回一个值，所以它也可以表示三元`cond ? A : B`语法在某些语言中可用。

Wasm 的控制流的其余部分基于`block`和`loop`块，以及`br`,`br_if`和`br_table`跳。反编译器会很好地接近这些构造，而不是试图推断它们可能来自的 while/for/switch 构造，因为这在优化输出时往往会更好地工作。例如，在`wasm-decompile`输出可能如下所示：

```c
loop A {
  // body of the loop here.
  if (cond) continue A;
}
```

这里`A`是一个标签，允许嵌套其中的多个标签。有一个`if`和`continue`与 while 循环相比，控制环路可能看起来有点陌生，但它直接对应于 Wasm 的`br_if`.

块是相似的，但它们不是向后分支，而是向前分支：

```c
block {
  if (cond) break;
  // body goes here.
}
```

这实际上实现了一个 if-then。如果可能的话，反编译器的未来版本可能会将这些转换为实际的if-then。

Wasm最令人惊讶的控制结构是`br_table`，它实现了类似`switch`，但使用嵌套除外`block`s，这往往很难阅读。反编译器将这些压平以使其略微变平
更易于遵循，例如：

```c
br_table[A, B, C, ..D](a);
label A:
return 0;
label B:
return 1;
label C:
return 2;
label D:
```

这类似于`switch`上`a`跟`D`是默认情况。

### 其他有趣的功能

反编译器：

*   可以从调试或链接信息中提取名称，或自行生成名称。使用现有名称时，它具有特殊代码来简化C++名称损坏的符号。
*   已经支持多值提案，这使得将事物转换为表达式和语句变得更加困难。返回多个值时，将使用其他变量。
*   它甚至可以从*内容*的数据部分。
*   为所有 Wasm 节类型输出漂亮的声明，而不仅仅是代码。例如，它尝试通过在可能的情况下将数据部分输出为文本来使数据部分可读。
*   支持运算符优先级（大多数 C 样式语言通用），以减少`()`在常用表达式上。

### 局限性

从根本上说，反编译 Wasm 比 JVM 字节码更难。

后者是未优化的，因此相对忠实于原始代码的结构，即使可能缺少名称，也是指唯一的类而不仅仅是内存位置。

相比之下，大多数`.wasm`输出已通过LLVM进行了大量优化，因此经常失去其大部分原始结构。输出代码与程序员编写的内容非常不同。这使得Wasm的反编译器成为一个更大的挑战，但这并不意味着我们不应该尝试！

## 更多

查看更多内容的最佳方式当然是反编译您自己的 Wasm 项目！

此外，更深入的指南`wasm-decompile`是[这里](https://github.com/WebAssembly/wabt/blob/master/docs/decompiler.md).它的实现在源文件中，以`decompiler` [这里](https://github.com/WebAssembly/wabt/tree/master/src)（随意贡献一个PR，让它变得更好！一些测试用例，显示`.wat`反编译器是[这里](https://github.com/WebAssembly/wabt/tree/master/test/decompile).
