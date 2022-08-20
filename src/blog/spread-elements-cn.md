***

标题： “加速传播元素”
作者： 'Hai Dang & Georg Neis'
日期： 2018-12-04 16：57：21
标签：

*   ECMAScript
*   基准
    描述： 'V8 v7.2 / 显著加快了 Array.from（array） 以及 \[...spread]，用于数组、字符串、集合和映射。
    推文：“1070344545685118976”

***

在V8团队为期三个月的实习期间，Hai Dang致力于提高V8团队的性能`[...array]`,`[...string]`,`[...set]`,`[...map.keys()]`和`[...map.values()]`（当展开元素位于数组文本的开头时）。他甚至做了`Array.from(iterable)`速度也快得多。本文解释了他的更改的一些血腥细节，这些细节包含在从 v7.2 开始的 V8 中。

## 展开元素

展开元素是具有以下形式的数组文本的组件`...iterable`.它们是在ES2015中引入的，作为从可迭代对象创建数组的一种方式。例如，数组文本`[1, ...arr, 4, ...b]`创建一个数组，其第一个元素为`1`后跟数组的元素`arr`然后`4`，最后是数组的元素`b`:

```js
const a = [2, 3];
const b = [5, 6, 7];
const result = [1, ...a, 4, ...b];
// → [1, 2, 3, 4, 5, 6, 7]
```

作为另一个示例，可以展开任何字符串以创建其字符数组（Unicode 码位）：

```js
const str = 'こんにちは';
const result = [...str];
// → ['こ', 'ん', 'に', 'ち', 'は']
```

类似地，可以展开任何集合以创建其元素的数组，按广告顺序排序：

```js
const s = new Set();
s.add('V8');
s.add('TurboFan');
const result = [...s];
// → ['V8', 'TurboFan']
```

通常，展开元素语法`...x`在数组文本中假定`x`提供迭代器（可通过以下方式访问）`x[Symbol.iterator]()`).然后使用此迭代器获取要插入到结果数组中的元素。

扩展数组的简单用例`arr`放入新数组中，无需在之前或之后添加任何其他元素，`[...arr]`，被认为是一种简洁的、惯用的浅克隆方式`arr`在ES2015中。不幸的是，在V8中，这个成语的性能远远落后于ES5的对应物。Hai实习的目标是改变这种状况！

## 为什么（或曾经！）传播元素缓慢？

有许多方法可以对阵列进行浅克隆`arr`.例如，您可以使用`arr.slice()`或`arr.concat()`或`[...arr]`.或者，您可以编写自己的`clone`采用标准的函数`for`-循环：

```js
function clone(arr) {
  // Pre-allocate the correct number of elements, to avoid
  // having to grow the array.
  const result = new Array(arr.length);
  for (let i = 0; i < arr.length; i++) {
    result[i] = arr[i];
  }
  return result;
}
```

理想情况下，所有这些选项都具有相似的性能特征。不幸的是，如果你选择`[...arr]`在 V8 中，它是 （或*是*） 可能慢于`clone`!原因是V8本质上是转译的`[...arr]`进入如下所示的迭代：

```js
function(arr) {
  const result = [];
  const iterator = arr[Symbol.iterator]();
  const next = iterator.next;
  for ( ; ; ) {
    const iteratorResult = next.call(iterator);
    if (iteratorResult.done) break;
    result.push(iteratorResult.value);
  }
  return result;
}
```

此代码通常比`clone`有几个原因：

1.  它需要创建`iterator`在开始时通过加载和评估`Symbol.iterator`财产。
2.  它需要创建和查询`iteratorResult`每一步都有对象。
3.  它使`result`数组在迭代的每一步，通过调用`push`，从而反复重新分配后备存储。

使用这种实现的原因是，如前所述，不仅可以在数组上进行传播，而且实际上可以在任意上完成。*可迭代*对象，并且必须遵循[迭代协议](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols).尽管如此，V8 应该足够智能，能够识别被传播的对象是否是一个数组，以便它可以在较低级别执行元素提取，从而：

1.  避免创建迭代器对象，
2.  避免创建迭代器结果对象，以及
3.  避免连续增长，从而重新分配结果数组（我们提前知道元素的数量）。

我们实现了这个简单的想法[断续器](/blog/csa)为*快*数组，即具有六种最常见之一的数组[元素种类](/blog/elements-kinds).优化适用于[常见的真实场景](/blog/real-world-performance)其中，传播发生在数组文本的开头，例如`[...foo]`.如下图所示，这种新的快速路径在扩展长度为100，000的数组时，性能提高了大约3×，使其比手写快约25%`clone`圈。

![Performance improvement of spreading a fast array](/\_img/spread-elements/spread-fast-array.png)

：：：备注
**注意：**虽然此处未显示，但当展开元素后跟其他组件（例如`[...arr, 1, 2, 3]`），但当它们前面有其他人时（例如`[1, 2, 3, ...arr]`).
:::

## 小心翼翼地沿着那条快速的路径前进

这显然是一个令人印象深刻的加速，但我们必须非常小心何时采取这种快速路径是正确的：JavaScript允许程序员以各种方式修改对象（甚至数组）的迭代行为。由于展开元素被指定为使用迭代协议，因此我们需要确保此类修改得到尊重。我们通过在原始迭代机制发生突变时完全避免快速路径来做到这一点。例如，这包括如下情况。

### 有`Symbol.iterator`财产

通常，数组`arr`没有自己的[`Symbol.iterator`](https://tc39.es/ecma262/#sec-symbol.iterator)属性，因此在查找该符号时，它将在数组的原型上找到。在下面的示例中，通过定义`Symbol.iterator`物业直接在`arr`本身。修改后，向上查找`Symbol.iterator`上`arr`导致空迭代器，从而传播`arr`不产生任何元素，数组文本的计算结果为空数组。

```js
const arr = [1, 2, 3];
arr[Symbol.iterator] = function() {
  return { next: function() { return { done: true }; } };
};
const result = [...arr];
// → []
```

### 改 性`%ArrayIteratorPrototype%`

这`next`方法也可以直接修改[`%ArrayIteratorPrototype%`](https://tc39.es/ecma262/#sec-%arrayiteratorprototype%-object)，数组迭代器的原型（影响所有数组）。

```js
Object.getPrototypeOf([][Symbol.iterator]()).next = function() {
  return { done: true };
}
const arr = [1, 2, 3];
const result = [...arr];
// → []
```

## 处理*多孔*阵 列

在复制带有孔的数组时也需要格外小心，即像这样的数组`['a', , 'c']`缺少一些元素。通过遵循迭代协议来分散这样的数组不会保留孔，而是在相应的索引处用数组原型中的值填充它们。默认情况下，数组的原型中没有元素，这意味着任何孔都填充`undefined`.例如`[...['a', , 'c']]`计算结果为新数组`['a', undefined, 'c']`.

我们的快速路径足够智能，可以在这种默认情况下处理漏洞。它不是盲目地复制输入数组的后备存储，而是注意孔并注意将它们转换为`undefined`值。下图包含长度为 100，000 的输入数组的测量值，该数组仅包含（标记的）600 个整数 — 其余为孔。它表明，现在传播这样一个多孔数组的速度×比使用`clone`功能。（他们曾经大致相当，但这没有显示在图表中）。

请注意，虽然`slice`包含在此图中，与之比较是不公平的，因为`slice`对于多孔数组具有不同的语义：它保留了所有孔，因此要做的工作量要少得多。

![Performance improvement of spreading a holey array of integers (HOLEY_SMI_ELEMENTS)](/\_img/spread-elements/spread-holey-smi-array.png)

填充孔`undefined`我们的快速路径必须执行并不像听起来那么简单：它可能需要将整个数组转换为不同的元素类型。下图测量了这种情况。设置与上面相同，只是这次600个数组元素是未装箱的双精度，并且数组具有`HOLEY_DOUBLE_ELEMENTS`元素种类。由于此元素类型不能保存标记的值，例如`undefined`，传播涉及昂贵的元素类型转换，这就是为什么分数为`[...a]`远低于上图。尽管如此，它仍然比快得多。`clone(a)`.

![Performance improvement of spreading a holey array of doubles (HOLEY_DOUBLE_ELEMENTS)](/\_img/spread-elements/spread-holey-double-array.png)

## 扩展字符串、集合和映射

跳过迭代器对象并避免增长结果数组的想法同样适用于传播其他标准数据类型。事实上，我们为基元字符串、集合和映射实现了类似的快速路径，每次都注意在存在修改后的迭代行为时绕过它们。

关于集合，快速路径不仅支持直接传播集合（\[...set]），但也展开其键迭代器（`[...set.keys()]`） 及其值迭代器 （`[...set.values()]`).在我们的微基准测试中，这些操作现在比以前快了大约18×

地图的快速路径类似，但不支持直接传播地图（`[...map]`），因为我们认为这是一个不常见的操作。出于同样的原因，两条快速路径都不支持`entries()`迭 代。在我们的微观基准测试中，这些操作现在比以前快了大约14×

用于扩展字符串 （`[...string]`），我们测量了大约5×改进，如下图中的紫色和绿色线条所示。请注意，这甚至比 TurboFan 优化的循环（TurboFan 理解字符串迭代并可以为其生成优化的代码）更快，由蓝色和粉红色行表示。在每种情况下都有两个图的原因是微基准测试在两种不同的字符串表示形式（单字节字符串和双字节字符串）上运行。

![Performance improvement of spreading a string](/\_img/spread-elements/spread-string.png)

![Performance improvement of spreading a set with 100,000 integers (magenta, about 18×), shown here in comparison with a for-of loop (red)](/\_img/spread-elements/spread-set.png)

## 提高`Array.from`性能

幸运的是，我们的扩散元素的快速路径可以重复用于`Array.from`在以下情况下`Array.from`使用可迭代对象调用，而不使用映射函数，例如，`Array.from([1, 2, 3])`.重用是可能的，因为在这种情况下，行为`Array.from`与传播完全相同。它带来了巨大的性能改进，如下所示，对于具有100个双精度的数组。

![Performance improvement of Array.from(array) where array contains 100 doubles](/\_img/spread-elements/array-from-array-of-doubles.png)

## 结论

V8 v7.2 / Chrome 72 大大提高了展开元素在数组文本前面时的性能，例如`[...x]`或`[...x, 1, 2]`.此改进适用于分散数组、基元字符串、集合、映射键、映射值，以及 （ 通过扩展 ）`Array.from(x)`.
