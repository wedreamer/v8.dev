***

标题： '改进`DataView`在 V8' 中的性能
作者： '泰奥泰姆·格罗恩斯，<i lang="fr">le savant de Data-Vue</i>和 Benedikt Meurer （[@bmeurer](https://twitter.com/bmeurer)），专业表演伙伴
化身：

*   'benedikt-meurer'
    日期： 2018-09-18 11：20：37
    标签：
*   ECMAScript
*   基准
    描述： “V8 v6.9 弥合了 DataView 和等效 TypedArray 代码之间的性能差距，有效地使 DataView 可用于性能关键型实际应用程序。
    推文：“1041981091727466496”

***

[`DataView`s](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)是在JavaScript中进行低级内存访问的两种可能方法之一，另一种是[`TypedArray`s](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray).直到现在，`DataView`s 的优化程度远低于`TypedArray`，导致图形密集型工作负载或解码/编码二进制数据等任务的性能降低。其原因主要是历史选择，例如[asm.js](http://asmjs.org/)选择`TypedArray`s 而不是`DataView`因此，激励发动机专注于性能`TypedArray`s.

由于性能下降，Google Maps团队等JavaScript开发人员决定避免`DataView`和 依赖`TypedArray`相反，以增加代码复杂性为代价。本文解释了我们如何带来`DataView`性能与同等水平相当，甚至超过同等水平`TypedArray`代码[V8 v6.9](/blog/v8-release-69)，有效地使`DataView`可用于性能关键型实际应用程序。

## 背景

自 ES2015 推出以来，JavaScript 一直支持在原始二进制缓冲区中读取和写入数据，称为[`ArrayBuffer`s](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer).`ArrayBuffer`s 不能直接访问;相反，程序必须使用所谓的*数组缓冲区视图*对象可以是`DataView`阿尔纳`TypedArray`.

`TypedArray`s 允许程序以统一类型值数组的形式访问缓冲区，例如`Int16Array`阿尔纳`Float32Array`.

```js
const buffer = new ArrayBuffer(32);
const array = new Int16Array(buffer);

for (let i = 0; i < array.length; i++) {
  array[i] = i * i;
}

console.log(array);
// → [0, 1, 4, 9, 16, 25, 36, 49, 64, 81, 100, 121, 144, 169, 196, 225]
```

另一方面`DataView`s 允许更细粒度的数据访问。它们通过为每个数字类型提供专用的 getter 和 setter，让程序员选择从缓冲区读取和写入缓冲区的值的类型，从而使它们对于序列化数据结构非常有用。

```js
const buffer = new ArrayBuffer(32);
const view = new DataView(buffer);

const person = { age: 42, height: 1.76 };

view.setUint8(0, person.age);
view.setFloat64(1, person.height);

console.log(view.getUint8(0)); // Expected output: 42
console.log(view.getFloat64(1)); // Expected output: 1.76
```

此外`DataView`s 还允许选择数据存储的字节序，这在从外部源（如网络、文件或 GPU）接收数据时非常有用。

```js
const buffer = new ArrayBuffer(32);
const view = new DataView(buffer);

view.setInt32(0, 0x8BADF00D, true); // Little-endian write.
console.log(view.getInt32(0, false)); // Big-endian read.
// Expected output: 0x0DF0AD8B (233876875)
```

高效`DataView`实现长期以来一直是一个功能请求（请参阅[此错误报告](https://bugs.chromium.org/p/chromium/issues/detail?id=225811)从 5 年前开始），我们很高兴地宣布，DataView 的性能现在不相上下！

## 旧版运行时实现

直到最近，`DataView`方法过去在 V8 中作为内置C++运行时函数实现。这是非常昂贵的，因为每次调用都需要从JavaScript到C++（以及返回）的昂贵过渡。

为了调查此实现所产生的实际性能成本，我们设置了一个性能基准，用于比较本机`DataView`使用 JavaScript 包装器模拟的 getter 实现`DataView`行为。此包装器使用`Uint8Array`从基础缓冲区逐个字节读取数据，然后计算这些字节的返回值。例如，下面是用于读取小端 32 位无符号整数值的函数：

```js
function LittleEndian(buffer) { // Simulate little-endian DataView reads.
  this.uint8View_ = new Uint8Array(buffer);
}

LittleEndian.prototype.getUint32 = function(byteOffset) {
  return this.uint8View_[byteOffset] |
    (this.uint8View_[byteOffset + 1] << 8) |
    (this.uint8View_[byteOffset + 2] << 16) |
    (this.uint8View_[byteOffset + 3] << 24);
};
```

`TypedArray`s 已经在 V8 中进行了大量优化，因此它们代表了我们想要匹配的性能目标。

![Original DataView performance](/\_img/dataview/dataview-original.svg)

我们的基准测试表明，原生`DataView`格特性能与**4 次**慢于`Uint8Array`基于-的包装器，用于大端和小端读取。

## 提高基准性能

我们提高性能的第一步`DataView`对象是将实现从C++运行时移动到[`CodeStubAssembler`（也称为 CSA）](/blog/csa).CSA是一种可移植的汇编语言，它允许我们直接在TurboFan的机器级中间表示（IR）中编写代码，我们用它来实现V8的JavaScript标准库的优化部分。在 CSA 中重写代码可以完全绕过C++调用，还可以利用 TurboFan 的后端生成高效的机器代码。

但是，手动编写 CSA 代码很麻烦。CSA 中的控制流表示方式与在程序集中非常相似，使用显式标签和`goto`s，这使得代码更难一目了然地阅读和理解。

为了使开发人员更容易为 V8 中优化的 JavaScript 标准库做出贡献，并提高可读性和可维护性，我们开始设计一种名为 V8 的新语言。*力矩*，编译为 CSA。目标*力矩*是抽象出使 CSA 代码更难编写和维护的低级细节，同时保留相同的性能配置文件。

重写`DataView`代码是开始使用Torque编写新代码的绝佳机会，并帮助Torque开发人员提供了大量关于该语言的反馈。这是什么`DataView`的`getUint32()`方法看起来像，写在扭矩：

```torque
macro LoadDataViewUint32(buffer: JSArrayBuffer, offset: intptr,
                    requested_little_endian: bool,
                    signed: constexpr bool): Number {
  let data_pointer: RawPtr = buffer.backing_store;

  let b0: uint32 = LoadUint8(data_pointer, offset);
  let b1: uint32 = LoadUint8(data_pointer, offset + 1);
  let b2: uint32 = LoadUint8(data_pointer, offset + 2);
  let b3: uint32 = LoadUint8(data_pointer, offset + 3);
  let result: uint32;

  if (requested_little_endian) {
    result = (b3 << 24) | (b2 << 16) | (b1 << 8) | b0;
  } else {
    result = (b0 << 24) | (b1 << 16) | (b2 << 8) | b3;
  }

  return convert<Number>(result);
}
```

移动`DataView`扭矩的方法已经显示了一个**3×改进**在性能上，但并不完全匹配`Uint8Array`-基于包装器性能。

![Torque DataView performance](/\_img/dataview/dataview-torque.svg)

## 针对涡轮风扇进行优化

当JavaScript代码变得热时，我们使用TurboFan优化编译器对其进行编译，以便生成高度优化的机器代码，其运行效率高于解释字节码。

TurboFan的工作原理是将传入的JavaScript代码转换为内部图形表示（更准确地说，[“节点之海”](https://darksi.de/d.sea-of-nodes/)).它从与JavaScript操作和语义匹配的高级节点开始，并逐渐将它们细化为越来越低的节点，直到它最终生成机器代码。

特别是，函数调用，例如调用`DataView`方法，在内部表示为`JSCall`node，最终归结为生成的机器代码中的实际函数调用。

但是，TurboFan允许我们检查是否`JSCall`node 实际上是对已知函数的调用，例如其中一个内置函数，并将此节点内联到 IR 中。这意味着复杂的`JSCall`在编译时被表示函数的子图替换。这使得TurboFan能够在后续的传递中优化函数的内部，作为更广泛上下文的一部分，而不是单独进行，最重要的是摆脱代价高昂的函数调用。

![Initial TurboFan DataView performance](/\_img/dataview/dataview-turbofan-initial.svg)

实施 TurboFan 内联最终使我们能够匹配甚至超越我们的性能`Uint8Array`包装器，并且是**8 次**与前者一样快，C++实施。

## 进一步的涡轮风扇优化

查看 TurboFan 在内联后生成的机器代码`DataView`方法上，还有一些改进的余地。这些方法的第一个实现试图非常严格地遵循标准，并在规范指示时抛出错误（例如，当尝试读取或写入底层的边界时）`ArrayBuffer`).

但是，我们在TurboFan中编写的代码旨在针对常见的热情况进行优化，以尽可能快地处理 - 它不需要支持所有可能的边缘情况。通过消除这些错误的所有复杂处理，并在我们需要抛出时取消优化回基线Torque实现，我们能够将生成的代码的大小减少约35%，从而产生非常明显的加速，以及相当简单的TurboFan代码。

为了遵循在 TurboFan 中尽可能专业化的想法，我们还删除了对 TurboFan 优化代码中太大（超出 Smi 范围）的索引或偏移量的支持。这使我们能够摆脱对不适合32位值的偏移所需的float64算术的处理，并避免在堆上存储大整数。

与最初的 TurboFan 实现相比，这增加了一倍多`DataView`基准测试分数。`DataView`s 现在的速度是`Uint8Array`包装，以及周围**速度快 16 倍**作为我们的原件`DataView`实现！

![Final TurboFan DataView performance](/\_img/dataview/dataview-turbofan-final.svg)

## 冲击

我们已经在我们自己的基准测试的基础上，在一些实际示例上评估了新实现的性能影响。

`DataView`s 通常用于解码从 JavaScript 以二进制格式编码的数据。一种这样的二进制格式是[断续器](https://en.wikipedia.org/wiki/FBX)，一种用于交换 3D 动画的格式。我们已经检测了流行的FBX加载器[三.js](https://threejs.org/)JavaScript 3D库，并测量其执行时间缩短了10%（约80毫秒）。

我们比较了`DataView`s 反对`TypedArray`s.我们发现我们的新`DataView`实现提供的性能几乎与`TypedArray`s 在访问以本机字节序（英特尔处理器上的小端）对齐的数据时，弥合大部分性能差距并使`DataView`是 V8 中的实用选择。

![DataView vs. TypedArray peak performance](/\_img/dataview/dataview-vs-typedarray.svg)

我们希望您现在能够开始使用`DataView`s，而不是依赖`TypedArray`垫片。请将您的反馈发送给我们`DataView`使用！您可以联系我们[通过我们的错误跟踪器](https://crbug.com/v8/new)，通过邮件发送至<v8-users@googlegroups.com>或通过[推特上的@v8js](https://twitter.com/v8js).
