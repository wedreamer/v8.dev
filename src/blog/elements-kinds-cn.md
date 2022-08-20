***

标题： '<i>元素种类</i>在 V8' 中
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2017-09-12 13：33：37
    标签：
*   内部
*   介绍
    描述： “这个技术深入探讨了 V8 如何在幕后优化阵列上的操作，以及这对 JavaScript 开发人员意味着什么。
    推文：“907608362191376384”

***

：：：备注
**注意：**如果您更喜欢观看演示文稿而不是阅读文章，那么请欣赏下面的视频！
:::

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/m9cTaYI95Zc" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

JavaScript 对象可以具有与之关联的任意属性。对象属性的名称可以包含任何字符。JavaScript引擎可以选择优化的一个有趣的情况是名称纯数字的属性，最具体地说是[数组索引](https://tc39.es/ecma262/#array-index).

在 V8 中，具有整数名称的属性 — 其最常见形式是由`Array`构造函数 — 是专门处理的。尽管在许多情况下，这些数字索引属性的行为与其他属性一样，但出于优化目的，V8 选择将它们与非数字属性分开存储。在内部，V8 甚至为这些属性指定了一个特殊名称：*元素*.对象具有[性能](/blog/fast-properties)映射到值，而数组具有映射到元素的索引。

虽然这些内部结构从不直接向JavaScript开发人员公开，但它们解释了为什么某些代码模式比其他代码模式更快。

## 常见元素种类

在运行 JavaScript 代码时，V8 会跟踪每个数组包含的元素类型。此信息允许 V8 专门针对此类元素优化阵列上的任何操作。例如，当您致电`reduce`,`map`或`forEach`在数组上，V8 可以根据数组包含的元素类型来优化这些操作。

以这个数组为例：

```js
const array = [1, 2, 3];
```

它包含哪些类型的元素？如果你问`typeof`运算符，它会告诉您数组包含`number`s.在语言层面上，这就是你得到的：JavaScript不区分整数，浮点数和双精度值 - 它们都只是数字。但是，在发动机级别，我们可以进行更精确的区分。此数组的元素类型为`PACKED_SMI_ELEMENTS`.在 V8 中，术语 Smi 是指用于存储小整数的特定格式。（我们将转到`PACKED`一分钟后部分。

稍后，将浮点数添加到同一数组会将其转换为更通用的元素类型：

```js
const array = [1, 2, 3];
// elements kind: PACKED_SMI_ELEMENTS
array.push(4.56);
// elements kind: PACKED_DOUBLE_ELEMENTS
```

将字符串文本添加到数组中会再次更改其元素类型。

```js
const array = [1, 2, 3];
// elements kind: PACKED_SMI_ELEMENTS
array.push(4.56);
// elements kind: PACKED_DOUBLE_ELEMENTS
array.push('x');
// elements kind: PACKED_ELEMENTS
```

到目前为止，我们已经看到了三种不同的元素类型，具有以下基本类型：

*   <b>断续器</b>都<b>我</b>ntegers，也被称为Smi。
*   双精度，表示不能表示为 Smi 的浮点数和整数。
*   常规元素，用于不能表示为 Smi 或双精度值的值。

请注意，双精度构成了 Smi 的更一般变体，规则元素是双精度值之上的另一种推广。可以表示为 Smi 的数字集是可以表示为双精度的数字的子集。

这里重要的是，元素类型的过渡只朝着一个方向发展：从特定的（例如`PACKED_SMI_ELEMENTS`） 到更一般（例如`PACKED_ELEMENTS`).一旦数组被标记为`PACKED_ELEMENTS`，则无法返回到`PACKED_DOUBLE_ELEMENTS`例如。

到目前为止，我们已经学到了以下内容：

*   V8 为每个数组分配一个元素种类。
*   数组的元素类型不是一成不变的 - 它可以在运行时更改。在前面的示例中，我们从`PACKED_SMI_ELEMENTS`自`PACKED_ELEMENTS`.
*   元素种类的过渡只能从特定种类过渡到更一般的种类。

## `PACKED`与。`HOLEY`种

到目前为止，我们只处理密集或打包的数组。在数组中创建孔（即使数组稀疏）会将元素种类降级为其“多孔”变体：

```js
const array = [1, 2, 3, 4.56, 'x'];
// elements kind: PACKED_ELEMENTS
array.length; // 5
array[9] = 1; // array[5] until array[8] are now holes
// elements kind: HOLEY_ELEMENTS
```

V8 之所以有这种区分，是因为与多孔阵列上的操作相比，可以更积极地优化打包阵列上的操作。对于打包阵列，可以有效地执行大多数操作。相比之下，对多孔数组的操作需要在原型链上进行额外的检查和昂贵的查找。

到目前为止，我们看到的每种基本元素（即Smis，doubles和常规元素）都有两种类型：包装和多孔版本。我们不仅可以从，比如说，`PACKED_SMI_ELEMENTS`自`PACKED_DOUBLE_ELEMENTS`，我们也可以从任何`PACKED`善良到它的`HOLEY`对方。

回顾一下：

*   最常见的元素种类进来`PACKED`和`HOLEY`口味。
*   对打包阵列的操作比对多孔数组的操作更有效。
*   元素种类可以从`PACKED`自`HOLEY`口味。

## 元素种类格子

V8 将此标签转换系统实现为[格子](https://en.wikipedia.org/wiki/Lattice\_%28order%29).下面是一个简化的可视化，仅包含最常见的元素类型：

![](/\_img/elements-kinds/lattice.svg)

只有通过晶格向下过渡才有可能。将单个浮点数添加到 Smis 数组后，即使以后用 Smi 覆盖浮点数，它也会被标记为 DOUBLE。同样，一旦在数组中创建了一个孔，它就会永远标记为有孔，即使你以后填充它。

V8 目前与众不同[21种不同的元素种类](https://cs.chromium.org/chromium/src/v8/src/elements-kind.h?l=14\&rcl=ec37390b2ba2b4051f46f153a8cc179ed4656f5d)，每个都有自己的一组可能的优化。

通常，更具体的元素种类可实现更细粒度的优化。元素种类在晶格中越往下，该对象的操作可能就越慢。为了获得最佳性能，请避免不必要地过渡到不太具体的类型 - 坚持使用适用于您的情况的最具体的类型。

## 性能提示

在大多数情况下，元素类跟踪在引擎盖下无形地工作，您无需担心。但是，您可以采取以下几项措施，从系统中获得最大的收益。

### 避免读取超出数组长度的读数

有点出乎意料的是（鉴于这篇文章的标题），我们的#1性能提示与元素类跟踪没有直接关系（尽管引擎盖下发生的事情有点相似）。超出数组长度的读取可能会产生令人惊讶的性能影响，例如读取`array[42]`什么时候`array.length === 5`.在本例中，数组索引`42`超出范围，该属性不存在于数组本身，因此 JavaScript 引擎必须执行昂贵的原型链查找。一旦负载遇到这种情况，V8 会记住“这个负载需要处理特殊情况”，并且它再也不会像读取越界之前那样快。

不要像这样编写循环：

```js
// Don’t do this!
for (let i = 0, item; (item = items[i]) != null; i++) {
  doSomething(item);
}
```

此代码读取数组中的所有元素，然后读取另一个元素。它只有在找到一个`undefined`或`null`元素。（jQuery 在一些地方使用了这种模式。

相反，以老式的方式编写循环，并不断迭代，直到你击中最后一个元素。

```js
for (let index = 0; index < items.length; index++) {
  const item = items[index];
  doSomething(item);
}
```

当您循环访问的集合是可迭代的（就像数组和`NodeList`s），那就更好了：只需使用`for-of`.

```js
for (const item of items) {
  doSomething(item);
}
```

特别是对于数组，您可以使用`forEach`内置：

```js
items.forEach((item) => {
  doSomething(item);
});
```

如今，两者的表现`for-of`和`forEach`与老式的不相上下`for`圈。

避免读取超过数组的长度！在这种情况下，V8 的边界检查失败，查看属性是否存在的检查失败，然后 V8 需要查找原型链。当您在计算中意外使用该值时，影响甚至更糟，例如：

```js
function Maximum(array) {
  let max = 0;
  for (let i = 0; i <= array.length; i++) { // BAD COMPARISON!
    if (array[i] > max) max = array[i];
  }
  return max;
}
```

在这里，最后一次迭代读取超过数组的长度，返回`undefined`，这不仅影响了负载，还污染了比较：现在它必须处理特殊情况，而不是仅比较数字。将终止条件固定为适当的`i < array.length`生成一个**6×**此示例的性能改进（在具有 10，000 个元素的数组上测量，因此迭代次数仅减少 0.01%）。

### 避免元素种类转换

一般来说，如果你需要对一个数组执行大量的操作，试着坚持使用尽可能具体的元素类型，这样 V8 就可以尽可能地优化这些操作。

这比看起来更难。例如，只需添加`-0`到一个小整数数组就足以将其转换为`PACKED_DOUBLE_ELEMENTS`.

```js
const array = [3, 2, 1, +0];
// PACKED_SMI_ELEMENTS
array.push(-0);
// PACKED_DOUBLE_ELEMENTS
```

因此，此阵列上的任何未来操作都将以与 Smis 完全不同的方式进行优化。

避免`-0`，除非您明确需要区分`-0`和`+0`在代码中。（你可能没有。

同样的事情也适用于`NaN`和`Infinity`.它们表示为双打，因此添加单个`NaN`或`Infinity`到数组`SMI_ELEMENTS`将其转换为`DOUBLE_ELEMENTS`.

```js
const array = [3, 2, 1];
// PACKED_SMI_ELEMENTS
array.push(NaN, Infinity);
// PACKED_DOUBLE_ELEMENTS
```

如果您计划对整数数组执行大量操作，请考虑规范化`-0`和阻止`NaN`和`Infinity`初始化值时。这样，数组将粘附在`PACKED_SMI_ELEMENTS`类。这种一次性规范化成本值得以后的优化。

事实上，如果你正在对一个数字数组进行数学运算，请考虑使用TypedArray。我们也有专门的元素类型。

### 优先使用数组而不是类似数组的对象

JavaScript中的一些对象 - 特别是在DOM中 - 看起来像数组，尽管它们不是正确的数组。可以自己创建类似数组的对象：

```js
const arrayLike = {};
arrayLike[0] = 'a';
arrayLike[1] = 'b';
arrayLike[2] = 'c';
arrayLike.length = 3;
```

此对象有一个`length`并支持索引元素访问（就像数组一样！），但它缺少数组方法，例如`forEach`在其原型上。不过，仍然可以在它上面调用数组泛型：

```js
Array.prototype.forEach.call(arrayLike, (value, index) => {
  console.log(`${ index }: ${ value }`);
});
// This logs '0: a', then '1: b', and finally '2: c'.
```

此代码调用`Array.prototype.forEach`内置于类似数组的对象上，并且按预期工作。但是，这比调用速度慢`forEach`在适当的阵列上，这在 V8 中得到了高度优化。如果您计划在此对象上多次使用数组内置功能，请考虑事先将其转换为实际的数组：

```js
const actualArray = Array.prototype.slice.call(arrayLike, 0);
actualArray.forEach((value, index) => {
  console.log(`${ index }: ${ value }`);
});
// This logs '0: a', then '1: b', and finally '2: c'.
```

一次性转换成本可能值得以后的优化，特别是如果您计划在阵列上执行大量操作。

这`arguments`例如，对象是一个类似数组的对象。可以在其上调用数组内置，但此类操作不会像正确数组那样完全优化。

```js
const logArgs = function() {
  Array.prototype.forEach.call(arguments, (value, index) => {
    console.log(`${ index }: ${ value }`);
  });
};
logArgs('a', 'b', 'c');
// This logs '0: a', then '1: b', and finally '2: c'.
```

ES2015 rest 参数可以在这里提供帮助。它们产生适当的数组，可以使用这些数组而不是类似数组`arguments`以优雅的方式对象。

```js
const logArgs = (...args) => {
  args.forEach((value, index) => {
    console.log(`${ index }: ${ value }`);
  });
};
logArgs('a', 'b', 'c');
// This logs '0: a', then '1: b', and finally '2: c'.
```

如今，没有充分的理由使用`arguments`对象直接。

通常，尽可能避免使用类似数组的对象，而改用适当的数组。

### 避免多态性

如果代码处理许多不同元素类型的数组，则可能导致多态操作比仅对单个元素类型操作的代码版本慢。

请考虑以下示例，其中使用各种元素类型调用库函数。（请注意，这不是本机`Array.prototype.forEach`，它在本文中讨论的元素特定于类型的优化之上有自己的一组优化。

```js
const each = (array, callback) => {
  for (let index = 0; index < array.length; ++index) {
    const item = array[index];
    callback(item);
  }
};
const doSomething = (item) => console.log(item);

each([], () => {});

each(['a', 'b', 'c'], doSomething);
// `each` is called with `PACKED_ELEMENTS`. V8 uses an inline cache
// (or “IC”) to remember that `each` is called with this particular
// elements kind. V8 is optimistic and assumes that the
// `array.length` and `array[index]` accesses inside the `each`
// function are monomorphic (i.e. only ever receive a single kind
// of elements) until proven otherwise. For every future call to
// `each`, V8 checks if the elements kind is `PACKED_ELEMENTS`. If
// so, V8 can re-use the previously-generated code. If not, more work
// is needed.

each([1.1, 2.2, 3.3], doSomething);
// `each` is called with `PACKED_DOUBLE_ELEMENTS`. Because V8 has
// now seen different elements kinds passed to `each` in its IC, the
// `array.length` and `array[index]` accesses inside the `each`
// function get marked as polymorphic. V8 now needs an additional
// check every time `each` gets called: one for `PACKED_ELEMENTS`
// (like before), a new one for `PACKED_DOUBLE_ELEMENTS`, and one for
// any other elements kinds (like before). This incurs a performance
// hit.

each([1, 2, 3], doSomething);
// `each` is called with `PACKED_SMI_ELEMENTS`. This triggers another
// degree of polymorphism. There are now three different elements
// kinds in the IC for `each`. For every `each` call from now on, yet
// another elements kind check is needed to re-use the generated code
// for `PACKED_SMI_ELEMENTS`. This comes at a performance cost.
```

内置方法（如`Array.prototype.forEach`） 可以更有效地处理这种多态性，因此请考虑在对性能敏感的情况下使用它们而不是 userland 库函数。

V8 中单态与多态的另一个示例涉及对象形状，也称为对象的隐藏类。要了解该案例，请查看[维亚切斯拉夫的文章](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html).

### 避免产生孔洞

对于实际的编码模式，访问多孔或打包数组之间的性能差异通常太小，甚至无法衡量。如果（这是一个很大的“如果”！）您的性能测量表明，在优化的代码中保存每一个机器指令都是值得的，那么您可以尝试将数组保持在打包元素模式下。假设我们正在尝试创建一个数组，例如：

```js
const array = new Array(3);
// The array is sparse at this point, so it gets marked as
// `HOLEY_SMI_ELEMENTS`, i.e. the most specific possibility given
// the current information.
array[0] = 'a';
// Hold up, that’s a string instead of a small integer… So the kind
// transitions to `HOLEY_ELEMENTS`.
array[1] = 'b';
array[2] = 'c';
// At this point, all three positions in the array are filled, so
// the array is packed (i.e. no longer sparse). However, we cannot
// transition to a more specific kind such as `PACKED_ELEMENTS`. The
// elements kind remains `HOLEY_ELEMENTS`.
```

一旦数组被标记为有孔，它就会永远保持多孔 - 即使它的所有元素稍后都存在！

创建数组的更好方法是改用文本：

```js
const array = ['a', 'b', 'c'];
// elements kind: PACKED_ELEMENTS
```

如果您事先不知道所有值，请创建一个空数组，稍后再创建`push`它的值。

```js
const array = [];
// …
array.push(someValue);
// …
array.push(someOtherValue);
```

此方法可确保数组永远不会转换为多孔元素类型。因此，V8 可能会为该阵列上的某些操作生成速度稍快的优化代码。

## 调试元素种类 { #debugging }

要找出给定对象的“元素类型”，请获取`d8`（由[从源代码构建](/docs/build)在调试模式下或通过抓取预编译的二进制文件使用[`jsvu`](https://github.com/GoogleChromeLabs/jsvu)），然后运行：

```bash
out/x64.debug/d8 --allow-natives-syntax
```

这将打开一个`d8`其中[特殊功能](https://cs.chromium.org/chromium/src/v8/src/runtime/runtime.h?l=20\&rcl=05720af2b09a18be5c41bbf224a58f3f0618f6be)如`%DebugPrint(object)`可用。其输出中的“elements”字段显示您传递给它的任何对象的“元素类型”。

```js
d8> const array = [1, 2, 3]; %DebugPrint(array);
DebugPrint: 0x1fbbad30fd71: [JSArray]
 - map = 0x10a6f8a038b1 [FastProperties]
 - prototype = 0x1212bb687ec1
 - elements = 0x1fbbad30fd19 <FixedArray[3]> [PACKED_SMI_ELEMENTS (COW)]
 - length = 3
 - properties = 0x219eb0702241 <FixedArray[0]> {
    #length: 0x219eb0764ac9 <AccessorInfo> (const accessor descriptor)
 }
 - elements= 0x1fbbad30fd19 <FixedArray[3]> {
           0: 1
           1: 2
           2: 3
 }
[…]
```

请注意，“COW”代表[写入时复制](https://en.wikipedia.org/wiki/Copy-on-write)，这是又一次内部优化。现在不要担心 - 这是另一篇博客文章的主题！

调试版本中提供的另一个有用标志是`--trace-elements-transitions`.启用它，让 V8 在发生任何元素类型转换时通知您。

```bash
$ cat my-script.js
const array = [1, 2, 3];
array[3] = 4.56;

$ out/x64.debug/d8 --trace-elements-transitions my-script.js
elements transition [PACKED_SMI_ELEMENTS -> PACKED_DOUBLE_ELEMENTS] in ~+34 at x.js:2 for 0x1df87228c911 <JSArray[3]> from 0x1df87228c889 <FixedArray[3]> to 0x1df87228c941 <FixedDoubleArray[22]>
```
