***

标题： “React 中 V8 性能悬崖的故事”
作者： 'Benedikt Meurer （[@bmeurer](https://twitter.com/bmeurer)）和马蒂亚斯·拜恩斯（[@mathias](https://twitter.com/mathias))'
化身：

*   'benedikt-meurer'
*   'mathias-bynens'
    发布日期： 2019-08-28 16：45：00
    标签：
*   内部
*   介绍
    描述： “本文描述了 V8 如何为各种 JavaScript 值选择最佳的内存中表示形式，以及这如何影响形状机制 — 所有这些都有助于解释 React 核心中最近的 V8 性能悬崖。
    推文：“1166723359696130049”

***

[以前](https://mathiasbynens.be/notes/shapes-ics)，我们讨论了 JavaScript 引擎如何通过使用 Shapes 和 Inline Caches 来优化对象和数组访问，并且我们已经探索了[引擎如何加快原型属性访问速度](https://mathiasbynens.be/notes/prototypes)特别。本文介绍了 V8 如何为各种 JavaScript 值选择最佳的内存中表示形式，以及它如何影响形状机制 — 所有这些都有助于解释[React 核心中最近的 V8 性能悬崖](https://github.com/facebook/react/issues/14365).

：：：备注
**注意：**如果您更喜欢观看演示文稿而不是阅读文章，那么请欣赏下面的视频！如果没有，请跳过视频并继续阅读。
:::

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/0I0d8LkDqyc" width="640" height="360" loading="lazy" allowfullscreen></iframe>
  </div>
  <figcaption><a href="https://www.youtube.com/watch?v=0I0d8LkDqyc">“JavaScript engine fundamentals: the good, the bad, and the ugly”</a> as presented by Mathias Bynens and Benedikt Meurer at AgentConf 2019.</figcaption>
</figure>

## JavaScript 类型

每个 JavaScript 值恰好有八种不同类型的一种：（目前）：`Number`,`String`,`Symbol`,`BigInt`,`Boolean`,`Undefined`,`Null`和`Object`.

![](/\_img/react-cliff/01-javascript-types.svg)

除了一个值得注意的例外，这些类型在JavaScript中可以通过`typeof`算子：

```js
typeof 42;
// → 'number'
typeof 'foo';
// → 'string'
typeof Symbol('bar');
// → 'symbol'
typeof 42n;
// → 'bigint'
typeof true;
// → 'boolean'
typeof undefined;
// → 'undefined'
typeof null;
// → 'object' 🤔
typeof { x: 42 };
// → 'object'
```

`typeof null`返回`'object'`，而不是`'null'`尽管`Null`是它自己的一种类型。要了解原因，请考虑所有JavaScript类型的集合分为两组：

*   *对象*（即`Object`类型）
*   *原*（即任何非对象值）

因此，`null`表示“无对象值”，而`undefined`意思是“没有价值”。

![](/\_img/react-cliff/02-primitives-objects.svg)

遵循这种思路，Brendan Eich 设计了 JavaScript 来制作`typeof`返回`'object'`对于右侧的所有值，即所有对象和`null`价值观，本着Java的精神。这就是为什么`typeof null === 'object'`尽管规格有一个单独的`Null`类型。

![](/\_img/react-cliff/03-primitives-objects-typeof.svg)

## 值表示

JavaScript 引擎必须能够在内存中表示任意的 JavaScript 值。但是，请务必注意，值的 JavaScript 类型与 JavaScript 引擎在内存中表示该值的方式是分开的。

价值`42`，例如，具有类型`number`在 JavaScript 中。

```js
typeof 42;
// → 'number'
```

有几种方法可以表示整数，例如`42`在内存中：

：：：表包装器
|代表|位 |
|--------------------------------- |--------------------------------------------------------------------------------- |
|二进制补码 8 位|`0010 1010`|
|二进制补码 32 位|`0000 0000 0000 0000 0000 0000 0010 1010`|
|压缩二进制编码十进制 （BCD） |`0100 0010`|
|32 位 IEEE-754 浮点|`0100 0010 0010 1000 0000 0000 0000 0000`|
|64 位 IEEE-754 浮点|`0100 0000 0100 0101 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000`|
:::

ECMAScript 将数字标准化为 64 位浮点值，也称为*双精度浮点*或*浮子64*.但是，这并不意味着JavaScript引擎一直以Float64表示形式存储数字 - 这样做将非常低效！引擎可以选择其他内部表示，只要可观察的行为与 Float64 完全匹配即可。

现实世界的JavaScript应用程序中的大多数数字恰好是[有效的 ECMAScript 数组索引](https://tc39.es/ecma262/#array-index)，即介于 0 到 2³²−2 之间的整数值。

```js
array[0]; // Smallest possible array index.
array[42];
array[2**32-2]; // Greatest possible array index.
```

JavaScript 引擎可以为此类数字选择最佳的内存中表示形式，以优化按索引访问数组元素的代码。要使处理器执行内存访问操作，数组索引必须在[二的补码](https://en.wikipedia.org/wiki/Two%27s_complement).相反，将数组索引表示为 Float64 会很浪费，因为每次有人访问数组元素时，引擎都必须在 Float64 和二进制补码之间来回转换。

32 位二进制的补码表示不仅对数组操作有用。通常**处理器执行整数运算的速度比浮点运算快得多**.这就是为什么在下一个示例中，第一个循环的速度是第二个循环的两倍。

```js
for (let i = 0; i < 1000; ++i) {
  // fast 🚀
}

for (let i = 0.1; i < 1000.1; ++i) {
  // slow 🐌
}
```

操作也是如此。下一段代码中模运算符的性能取决于您是否正在处理整数。

```js
const remainder = value % divisor;
// Fast 🚀 if `value` and `divisor` are represented as integers,
// slow 🐌 otherwise.
```

如果两个操作数都表示为整数，则 CPU 可以非常有效地计算结果。V8 具有额外的快速路径，适用于以下情况：`divisor`是两的幂。对于表示为浮点数的值，计算要复杂得多，并且需要更长的时间。

由于整数运算的执行速度通常比浮点运算快得多，因此引擎似乎总是对所有整数和整数运算的所有结果使用二进制补码。不幸的是，这将违反 ECMAScript 规范！ECMAScript 在 Float64 上标准化，依此类推**某些整数运算实际上会产生浮点数**.在这种情况下，JS引擎产生正确的结果非常重要。

```js
// Float64 has a safe integer range of 53 bits. Beyond that range,
// you must lose precision.
2**53 === 2**53+1;
// → true

// Float64 supports negative zeros, so -1 * 0 must be -0, but
// there’s no way to represent negative zero in two’s complement.
-1*0 === -0;
// → true

// Float64 has infinities which can be produced through division
// by zero.
1/0 === Infinity;
// → true
-1/0 === -Infinity;
// → true

// Float64 also has NaNs.
0/0 === NaN;
```

即使左侧的值是整数，右侧的所有值也是浮点数。这就是为什么使用32位二进制补码无法正确执行上述操作的原因。JavaScript 引擎必须特别注意确保整数运算适当地回退，以产生花哨的 Float64 结果。

对于 31 位有符号整数范围内的小整数，V8 使用一种特殊表示形式，称为`Smi`.任何不是`Smi`表示为`HeapObject`，这是内存中某个实体的地址。对于数字，我们使用一种特殊的`HeapObject`，即所谓的`HeapNumber`，表示不在`Smi`范围。

```js
 -Infinity // HeapNumber
-(2**30)-1 // HeapNumber
  -(2**30) // Smi
       -42 // Smi
        -0 // HeapNumber
         0 // Smi
       4.2 // HeapNumber
        42 // Smi
   2**30-1 // Smi
     2**30 // HeapNumber
  Infinity // HeapNumber
       NaN // HeapNumber
```

如上例所示，一些 JavaScript 数字表示为`Smi`s，而其他表示为`HeapNumber`s.V8 专门针对以下方面进行了优化：`Smi`s，因为小整数在现实世界的JavaScript程序中非常普遍。`Smi`不需要在内存中作为专用实体分配 ，并且通常启用快速整数操作。

这里重要的一点是**即使具有相同 JavaScript 类型的值也可以以完全不同的方式表示**幕后，作为优化。

### `Smi`与。`HeapNumber`与。`MutableHeapNumber`{ #smi-heapnumber-mutableheapnumber }

以下是它在引擎盖下的工作原理。假设您有以下对象：

```js
const o = {
  x: 42,  // Smi
  y: 4.2, // HeapNumber
};
```

价值`42`为`x`可以编码为`Smi`，因此它可以存储在对象本身的内部。价值`4.2`另一方面，需要一个单独的实体来保存值，并且对象指向该实体。

![](/\_img/react-cliff/04-smi-vs-heapnumber.svg)

现在，假设我们运行以下 JavaScript 代码段：

```js
o.x += 10;
// → o.x is now 52
o.y += 1;
// → o.y is now 5.2
```

在本例中，值`x`可以就地更新，因为新值`52`也适合`Smi`范围。

![](/\_img/react-cliff/05-update-smi.svg)

但是，新值`y=5.2`不适合`Smi`并且也与以前的值不同`4.2`，因此 V8 必须分配一个新的`HeapNumber`要将任务分配到的实体`y`.

![](/\_img/react-cliff/06-update-heapnumber.svg)

`HeapNumber`s 是不可变的，这可以实现某些优化。例如，如果我们分配`y`s 值`x`:

```js
o.x = o.y;
// → o.x is now 5.2
```

...我们现在可以链接到相同的`HeapNumber`而不是为相同的值分配一个新的。

![](/\_img/react-cliff/07-heapnumbers.svg)

一个缺点`HeapNumber`s 是不可变的，因为更新字段时，使用`Smi`范围通常，如以下示例所示：

```js
// Create a `HeapNumber` instance.
const o = { x: 0.1 };

for (let i = 0; i < 5; ++i) {
  // Create an additional `HeapNumber` instance.
  o.x += 1;
}
```

第一行将创建一个`HeapNumber`具有初始值的实例`0.1`.循环正文将此值更改为`1.1`,`2.1`,`3.1`,`4.1`，最后`5.1`，总共创建六个`HeapNumber`沿途的实例，循环完成后，其中五个是垃圾。

![](/\_img/react-cliff/08-garbage-heapnumbers.svg)

为避免此问题，V8 提供了一种更新非`Smi`数字字段也就地，作为优化。当数值字段保存`Smi`范围，V8 将该字段标记为`Double`字段在形状上，并分配一个所谓的`MutableHeapNumber`保存编码为 Float64 的实际值。

![](/\_img/react-cliff/09-mutableheapnumber.svg)

当字段的值更改时，V8 不再需要分配新的`HeapNumber`，但可以只更新`MutableHeapNumber`就地。

![](/\_img/react-cliff/10-update-mutableheapnumber.svg)

但是，这种方法也有一个陷阱。由于值`MutableHeapNumber`可以改变，重要的是这些不被传递。

![](/\_img/react-cliff/11-mutableheapnumber-to-heapnumber.svg)

例如，如果分配`o.x`到其他变量`y`，您不希望的值`y`更改下次`o.x`更改 - 这将违反JavaScript规范！因此，当`o.x`被访问，号码必须为*重新装箱*成为常规`HeapNumber`在将其分配给之前`y`.

对于花车，V8在幕后执行了上述所有“拳击”魔术。但是对于小整数，使用`MutableHeapNumber`方法，因为`Smi`是一种更有效的表示形式。

```js
const object = { x: 1 };
// → no “boxing” for `x` in object

object.x += 1;
// → update the value of `x` inside object
```

为了避免效率低下，对于小整数，我们所要做的就是将形状上的字段标记为`Smi`表示形式，只需更新数字值，只要它适合小整数范围即可。

![](/\_img/react-cliff/12-smi-no-boxing.svg)

## 形状弃用和迁移

那么，如果一个字段最初包含`Smi`，但后来在小整数范围之外保存一个数字？就像在这种情况下，两个对象都使用相同的形状，其中`x`表示为`Smi`最初：

```js
const a = { x: 1 };
const b = { x: 2 };
// → objects have `x` as `Smi` field now

b.x = 0.2;
// → `b.x` is now represented as a `Double`

y = a.x;
```

这从指向同一形状的两个对象开始，其中`x`标记为`Smi`表示法：

![](/\_img/react-cliff/13-shape.svg)

什么时候`b.x`更改为`Double`表示，V8 分配一个新形状，其中`x`已分配`Double`表示形式，并指向空形状。V8 还分配了一个`MutableHeapNumber`以保存新值`0.2`对于`x`财产。然后我们更新对象`b`指向此新形状，并将对象中的槽更改为指向先前分配的`MutableHeapNumber`偏移量为 0。最后，我们将旧形状标记为已弃用，并将其从过渡树中取消链接。这是通过对`'x'`从空形状到新创建的形状。

![](/\_img/react-cliff/14-shape-transition.svg)

此时，我们无法完全删除旧形状，因为它仍由`a`，并且遍历内存以查找指向旧形状的所有对象并热切地更新它们将过于昂贵。相反，V8 懒惰地执行此操作：任何属性访问或分配`a`首先将其迁移到新形状。我们的想法是最终使已弃用的形状无法访问，并让垃圾回收器将其删除。

![](/\_img/react-cliff/15-shape-deprecation.svg)

如果更改表示的字段是*不*链中的最后一个：

```js
const o = {
  x: 1,
  y: 2,
  z: 3,
};

o.y = 0.1;
```

在这种情况下，V8需要找到所谓的*分裂形状*，这是引入相关属性之前链中的最后一个形状。在这里，我们正在改变`y`，所以我们需要找到最后一个没有的形状`y`，在我们的示例中，它是引入的形状`x`.

![](/\_img/react-cliff/16-split-shape.svg)

从拆分形状开始，我们创建一个新的过渡链`y`它重播了所有以前的过渡，但`'y'`被标记为`Double`表示法。我们使用这个新的过渡链`y`，将旧子树标记为已弃用。在最后一步中，我们迁移实例`o`到新形状，使用`MutableHeapNumber`以保持`y`现在。这样，新对象就不会采用旧路径，并且一旦对旧形状的所有引用都消失，树中已弃用的形状部分就会消失。

## 可扩展性和完整性级别转换

`Object.preventExtensions()`防止将新属性添加到对象中。如果尝试，则会引发异常。（如果您未处于严格模式，则不会抛出，但会静默地执行任何操作。

```js
const object = { x: 1 };
Object.preventExtensions(object);
object.y = 2;
// TypeError: Cannot add property y;
//            object is not extensible
```

`Object.seal`执行与`Object.preventExtensions`，但它也将所有属性标记为不可配置，这意味着您无法删除它们，也无法更改它们的可枚举性、可配置性或可写性。

```js
const object = { x: 1 };
Object.seal(object);
object.y = 2;
// TypeError: Cannot add property y;
//            object is not extensible
delete object.x;
// TypeError: Cannot delete property x
```

`Object.freeze`执行与`Object.seal`，但它也通过标记现有属性不可写来防止更改这些属性的值。

```js
const object = { x: 1 };
Object.freeze(object);
object.y = 2;
// TypeError: Cannot add property y;
//            object is not extensible
delete object.x;
// TypeError: Cannot delete property x
object.x = 3;
// TypeError: Cannot assign to read-only property x
```

让我们考虑这个具体的例子，有两个对象，它们都有一个属性。`x`，然后我们阻止对第二个对象的任何进一步扩展。

```js
const a = { x: 1 };
const b = { x: 2 };

Object.preventExtensions(b);
```

它开始就像我们已经知道的那样，从空形状过渡到持有属性的新形状。`'x'`（表示为`Smi`).当我们阻止扩展到`b`，我们将执行到标记为不可扩展的新形状的特殊过渡。这种特殊的过渡不会引入任何新的属性 - 它实际上只是一个标记。

![](/\_img/react-cliff/17-shape-nonextensible.svg)

请注意，我们不能只用`x`就地，因为另一个对象需要它`a`，这仍然是可扩展的。

## React 性能问题 { #react }

让我们把它们放在一起，用我们学到的东西来理解[最近的 React 问题 #14365](https://github.com/facebook/react/issues/14365).当 React 团队分析一个真实世界的应用程序时，他们发现了一个奇怪的 V8 性能悬崖，影响了 React 的核心。下面是该 Bug 的简化重现：

```js
const o = { x: 1, y: 2 };
Object.preventExtensions(o);
o.y = 0.2;
```

我们有一个对象，其中包含两个字段，这些字段具有`Smi`表示法。我们阻止对对象的任何进一步扩展，并最终强制第二个字段`Double`表示法。

正如我们之前所学到的，这大致创建了以下设置：

![](/\_img/react-cliff/18-repro-shape-setup.svg)

这两个属性都标记为`Smi`表示形式，最后的过渡是扩展性过渡，用于将形状标记为不可扩展。

现在我们需要改变`y`自`Double`表示，这意味着我们需要再次从查找拆分形状开始。在本例中，是形状引入了`x`.但现在V8感到困惑，因为分裂的形状是可扩展的，而当前的形状被标记为不可扩展的。在这种情况下，V8并不真正知道如何正确重放过渡。因此，V8 基本上放弃了试图理解这一点，而是创建了一个单独的形状，该形状不连接到现有的形状树，也不与任何其他对象共享。将其视为*孤立形状*:

![](/\_img/react-cliff/19-orphaned-shape.svg)

你可以想象，如果这种情况发生在很多物体上，那就太糟糕了，因为这会使整个形状系统变得毫无用处。

在 React 的例子中，下面是发生了什么：每个`FiberNode`有几个字段，这些字段应该在打开分析时保存时间戳。

```js
class FiberNode {
  constructor() {
    this.actualStartTime = 0;
    Object.preventExtensions(this);
  }
}

const node1 = new FiberNode();
const node2 = new FiberNode();
```

这些字段（如`actualStartTime`） 初始化为`0`或`-1`，因此从`Smi`表示法。但后来，实际的浮点时间戳来自[`performance.now()`](https://w3c.github.io/hr-time/#dom-performance-now)存储在这些字段中，导致它们转到`Double`表示，因为它们不适合`Smi`.最重要的是，React 还阻止了对`FiberNode`实例。

最初，上面的简化示例如下所示：

![](/\_img/react-cliff/20-fibernode-shape.svg)

有两个实例共享一个形状树，所有实例都按预期工作。但是，当您存储实时时间戳时，V8 会困惑地找到拆分形状：

![](/\_img/react-cliff/21-orphan-islands.svg)

V8 将新的孤立形状分配给`node1`，同样的事情发生在`node2`一段时间后，导致两个*孤儿岛屿*，每个都有自己的不相交形状。许多现实世界的 React 应用程序不仅有两个，而是有数以万计的这些应用程序。`FiberNode`s.可以想象，这种情况对于V8的性能来说并不是特别好。

幸[我们已经修复了这个性能悬崖](https://chromium-review.googlesource.com/c/v8/v8/+/1442640/)在[V8 v7.4](/blog/v8-release-74)，而我们是[希望使现场表示的变化更便宜](https://bit.ly/v8-in-place-field-representation-changes)以移除任何剩余的性能悬崖。修复后，V8 现在执行了正确的操作：

![](/\_img/react-cliff/22-fix.svg)

二`FiberNode`实例指向不可扩展的形状，其中`'actualStartTime'`是一个`Smi`田。当第一次分配到`node1.actualStartTime`发生时，将创建新的转换链，并将以前的链标记为已弃用：

![](/\_img/react-cliff/23-fix-fibernode-shape-1.svg)

请注意，扩展性转换现在如何在新链中正确重放。

![](/\_img/react-cliff/24-fix-fibernode-shape-2.svg)

分配到`node2.actualStartTime`，两个节点都引用新形状，并且垃圾回收器可以清理过渡树中已弃用的部分。

：：：备注
**注意：**你可能会认为所有这些形状的弃用/迁移都很复杂，你是对的。事实上，我们怀疑在现实世界的网站上，它会导致更多的问题（在性能，内存使用和复杂性方面），而不是帮助，特别是因为[指针压缩](https://bugs.chromium.org/p/v8/issues/detail?id=7703)我们将无法再使用它来在对象中以内联方式存储双值字段。所以，我们希望[完全删除 V8 的形状弃用机制](https://bugs.chromium.org/p/v8/issues/detail?id=9606).你可以说它是*\*戴上太阳镜\**已弃用。*哎呀...*
:::

React 团队[缓解了问题](https://github.com/facebook/react/pull/14383)通过确保所有时间和持续时间字段`FiberNode`s 开头为`Double`表示法：

```js
class FiberNode {
  constructor() {
    // Force `Double` representation from the start.
    this.actualStartTime = Number.NaN;
    // Later, you can still initialize to the value you want:
    this.actualStartTime = 0;
    Object.preventExtensions(this);
  }
}

const node1 = new FiberNode();
const node2 = new FiberNode();
```

而不是`Number.NaN`，任何不符合`Smi`可以使用范围。示例包括`0.000001`,`Number.MIN_VALUE`,`-0`和`Infinity`.

值得指出的是，具体的 React bug 是特定于 V8 的，一般来说，开发人员不应该针对特定版本的 JavaScript 引擎进行优化。不过，当事情不起作用时，有一个手柄还是很好的。

请记住，JavaScript引擎在引擎盖下执行了一些魔术，如果可能的话，您可以通过不混合类型来帮助它。例如，不要使用`null`，因为这会禁用字段表示跟踪的所有好处，并且使您的代码更具可读性：

```js
// Don’t do this!
class Point {
  x = null;
  y = null;
}

const p = new Point();
p.x = 0.1;
p.y = 402;
```

换句话说，**编写可读的代码，性能将随之而来！**

## 外卖 { #takeaways }

在这次深入探讨中，我们介绍了以下内容：

*   JavaScript区分“原语”和“对象”，以及`typeof`是个骗子。
*   即使具有相同 JavaScript 类型的值在后台也可能具有不同的表示形式。
*   V8 尝试为 JavaScript 程序中的每个属性找到最佳表示形式。
*   我们已经讨论了 V8 如何处理形状弃用和迁移，包括可扩展性转换。

基于这些知识，我们确定了一些实用的JavaScript编码技巧，可以帮助提高性能：

*   始终以相同的方式初始化对象，以便形状有效。
*   为字段选择合理的初始值，以帮助 JavaScript 引擎进行表示选择。
