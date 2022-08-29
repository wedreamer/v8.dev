***

标题： “在 V8 中对事物进行排序”
作者： '西蒙·尊德 （[@nimODota](https://twitter.com/nimODota)），一致的比较器'
化身：

*   西蒙-祖恩德
    日期： 2018-09-28 11：20：37
    标签：
*   ECMAScript
*   内部
    描述： '从 V8 v7.0 / Chrome 70 开始，Array.prototype.sort 是稳定的。
    推文：“1045656758700650502”

***

`Array.prototype.sort`是 V8 中在自托管 JavaScript 中实现的最后一个内置组件之一。移植它为我们提供了尝试不同算法和实现策略的机会，最后[使其稳定](https://mathiasbynens.be/demo/sort-stability)在 V8 v7.0 / Chrome 70 中。

## 背景

在JavaScript中排序是很困难的。这篇博客文章探讨了排序算法和 JavaScript 语言之间交互中的一些怪癖，并描述了我们将 V8 迁移到稳定算法并使性能更可预测的旅程。

在比较不同的排序算法时，我们会查看它们最差和平均的性能，作为内存操作或比较次数的渐近增长（即“Big O”表示法）的边界。请注意，在动态语言（如 JavaScript）中，比较操作通常比内存访问昂贵一个数量级。这是因为在排序时比较两个值通常涉及对用户代码的调用。

让我们看一个简单的示例，根据用户提供的比较函数将一些数字按升序排序。一个*一致*比较函数返回`-1`（或任何其他负值），`0`或`1`（或任何其他正值）当提供的两个值分别小于、等于或大于时。不遵循此模式的比较函数是*不一致的*并且可能具有任意的副作用，例如修改它要排序的数组。

```js
const array = [4, 2, 5, 3, 1];

function compare(a, b) {
  // Arbitrary code goes here, e.g. `array.push(1);`.
  return a - b;
}

// A “typical” sort call.
array.sort(compare);
```

即使在下一个示例中，也可能发生对用户代码的调用。“默认”比较函数调用`toString`，并对字符串表示形式进行词典比较。

```js
const array = [4, 2, 5, 3, 1];

array.push({
  toString() {
    // Arbitrary code goes here, e.g. `array.push(1);`.
    return '42';
  }
});

// Sort without a comparison function.
array.sort();
```

### 通过访问器和原型链交互获得更多乐趣{ #accessors-原型 }

这是我们抛弃规范并冒险进入“实现定义”行为领域的部分。该规范有一个完整的条件列表，当满足这些条件时，允许引擎按照它认为合适的方式对对象/数组进行排序 - 或者根本不排序。发动机仍然必须遵循一些基本规则，但其他一切都悬而未决。一方面，这为引擎开发人员提供了尝试不同实现的自由。另一方面，用户期望一些合理的行为，即使规范不要求有任何行为。由于“合理行为”并不总是很容易确定，这一事实使情况进一步复杂化。

本节说明，还有一些方面`Array#sort`其中发动机行为差异很大。这些都是硬性情况，如上所述，并不总是清楚“正确的做法”到底是什么。我们*高度*建议不要像这样编写代码;引擎不会针对它进行优化。

第一个示例显示了一个数组，其中包含一些访问器（即 getter 和 setter）和不同 JavaScript 引擎中的“调用日志”。访问器是生成的排序顺序由实现定义的第一种情况：

```js
const array = [0, 1, 2];

Object.defineProperty(array, '0', {
  get() { console.log('get 0'); return 0; },
  set(v) { console.log('set 0'); }
});

Object.defineProperty(array, '1', {
  get() { console.log('get 1'); return 1; },
  set(v) { console.log('set 1'); }
});

array.sort();
```

下面是该代码段在各种引擎中的输出。请注意，这里没有“正确”或“错误”的答案 - 规范将此留给实现！

    // Chakra
    get 0
    get 1
    set 0
    set 1

    // JavaScriptCore
    get 0
    get 1
    get 0
    get 0
    get 1
    get 1
    set 0
    set 1

    // V8
    get 0
    get 0
    get 1
    get 1
    get 1
    get 0

    #### SpiderMonkey
    get 0
    get 1
    set 0
    set 1

下一个示例显示了与原型链的交互。为了简洁起见，我们不显示通话记录。

```js
const object = {
 1: 'd1',
 2: 'c1',
 3: 'b1',
 4: undefined,
 __proto__: {
   length: 10000,
   1: 'e2',
   10: 'a2',
   100: 'b2',
   1000: 'c2',
   2000: undefined,
   8000: 'd2',
   12000: 'XX',
   __proto__: {
     0: 'e3',
     1: 'd3',
     2: 'c3',
     3: 'b3',
     4: 'f3',
     5: 'a3',
     6: undefined,
   },
 },
};
Array.prototype.sort.call(object);
```

输出显示`object`排序后。同样，这里没有正确的答案。这个例子只是展示了索引属性和原型链之间的交互是多么奇怪：

```js
// Chakra
['a2', 'a3', 'b1', 'b2', 'c1', 'c2', 'd1', 'd2', 'e3', undefined, undefined, undefined]

// JavaScriptCore
['a2', 'a2', 'a3', 'b1', 'b2', 'b2', 'c1', 'c2', 'd1', 'd2', 'e3', undefined]

// V8
['a2', 'a3', 'b1', 'b2', 'c1', 'c2', 'd1', 'd2', 'e3', undefined, undefined, undefined]

// SpiderMonkey
['a2', 'a3', 'b1', 'b2', 'c1', 'c2', 'd1', 'd2', 'e3', undefined, undefined, undefined]
```

### V8 在排序之前和之后执行的操作 { #before排序 }

：：：备注
**注意：**此部分已于 2019 年 6 月更新，以反映以下方面的更改：`Array#sort`V8 v7.7 中的预处理和后处理。
:::

V8 在实际对任何内容进行排序之前有一个预处理步骤，还有一个后处理步骤。基本思想是收集所有非`undefined`值放入临时列表中，对此临时列表进行排序，然后将排序后的值写回实际的数组或对象中。这使 V8 在分拣过程中无需关心与访问器或原型链的交互。

规格预期`Array#sort`以生成在概念上可以分为三个段的排序顺序：

1.  所有非`undefined`值已排序 w.r.t. 到比较函数。
2.  都`undefined`s.
3.  所有孔，即不存在的属性。

实际的排序算法只需要应用于第一个段。为了实现这一点，V8有一个预处理步骤，大致如下：

1.  让`length`是的价值`”length”`要排序的数组或对象的属性。
2.  让`numberOfUndefineds`为 0。
3.  对于每个`value`在`[0, length)`:
    一个。如果`value`是一个洞：什么都不做
    b.如果`value`是`undefined`：增量`numberOfUndefineds`由 1.
    c. 否则添加`value`到临时列表`elements`.

执行这些步骤后，所有非`undefined`值包含在临时列表中`elements`.`undefined`s 只是简单地计数，而不是添加到`elements`.如上所述，规范要求`undefined`s 必须排序到最后。除了`undefined`值实际上并没有传递给用户提供的比较函数，因此我们可以只计算`undefined`发生了。

下一步是实际排序`elements`.看[关于TimSort的部分](/blog/array-sort#timsort)以获取详细说明。

排序完成后，必须将排序的值写回原始数组或对象。后处理步骤由处理概念段的三个阶段组成：

1.  写回来自`elements`到原始对象的范围内`[0, elements.length)`.
2.  设置以下位置的所有值`[elements.length, elements.length + numberOfUndefineds)`自`undefined`.
3.  删除范围中的所有值`[elements.length + numberOfUndefineds, length)`.

如果原始对象包含排序范围内的孔，则需要执行步骤 3。值在`[elements.length + numberOfUndefineds, length)`已移至前面，不执行步骤 3 将导致值重复。

## 历史

`Array.prototype.sort`和`TypedArray.prototype.sort`依赖于用 JavaScript 编写的相同 Quicksort 实现。排序算法本身相当简单：基础是快速排序，对于较短的数组（长度为 10 <），具有插入排序回退。当快速排序递归达到子数组长度 10 时，也使用了插入排序回退。插入排序对于较小的数组更有效。这是因为快速排序在分区后被递归调用两次。每个这样的递归调用都有创建（和丢弃）堆栈帧的开销。

选择合适的枢轴元素对快速排序有很大的影响。V8 采用了两种策略：

*   选择透视表作为排序的子数组的第一个、最后一个和第三个元素的中位数。对于较小的数组，第三个元素只是中间元素。
*   对于较大的数组，取一个样本，然后进行排序，排序样本的中位数作为上述计算中的第三个元素。

快速排序的优点之一是它可以就地排序。内存开销来自在对大型数组进行排序时为样本分配一个小数组，以及 log（n） 堆栈空间。缺点是它不是一个稳定的算法，并且该算法有可能遇到QuickSort降级为O（n²）的最坏情况。

### V8 扭矩简介

作为您可能听说过的V8博客的狂热读者[`CodeStubAssembler`](/blog/csa)或简称CSA。CSA是一个V8组件，它允许我们直接在C++中编写低级TurboFan IR，然后使用TurboFan的后端将其转换为相应架构的机器代码。

CSA被大量用于为JavaScript内置编写所谓的“快速路径”。内置的快速路径版本通常检查某些不变量是否成立（例如，原型链上没有元素，没有访问器等），然后使用更快，更具体的操作来实现内置功能。这可能导致执行时间比更通用的版本快一个数量级。

CSA的缺点是它确实可以被认为是一种汇编语言。使用显式对控制流进行建模`labels`和`gotos`，这使得在 CSA 中实现更复杂的算法变得难以阅读且容易出错。

进入[V8 扭矩](/docs/torque).Torque是一种特定于领域的语言，具有类似TypeScript的语法，目前使用CSA作为其唯一的编译目标。扭矩允许与 CSA 几乎相同的控制水平，同时提供更高级别的结构，例如`while`和`for`循环。此外，它是强类型的，将来将包含安全检查，例如自动越界检查，为V8工程师提供更强大的保证。

在V8 Torque中重写的第一个主要内置是[`TypedArray#sort`](/blog/v8-release-68)和[`Dataview`操作](/blog/dataview).两者都有一个额外的目的，即向Torque开发人员提供反馈，说明需要哪些语言功能，并且应该使用习语来有效地编写内置组件。在撰写本文时，有几个`JSArray`内置的自托管JavaScript回退实现被转移到Torque（例如`Array#unshift`），而其他的则被完全重写（例如`Array#splice`和`Array#reverse`).

### 移动`Array#sort`到扭矩

初始`Array#sort`Torque版本或多或少是JavaScript实现的直接端口。唯一的区别是，不是对较大的数组使用采样方法，而是随机选择用于透视计算的第三个元素。

这工作得相当不错，但由于它仍然使用Quicksort，`Array#sort`仍然不稳定。[对稳定版的要求`Array#sort`](https://bugs.chromium.org/p/v8/issues/detail?id=90)是V8错误跟踪器中最古老的门票之一。将Timsort作为下一步进行实验为我们提供了许多东西。首先，我们喜欢它是稳定的，并提供一些很好的算法保证（见下一节）。其次，Torque仍然是一个正在进行的工作，并实现了更复杂的内置功能，例如`Array#sort`与Timsort一起产生了许多可操作的反馈，影响了Torque作为一种语言。

## 蒂姆索特

Timsort最初由Tim Peters于2002年为Python开发，可以最好地描述为自适应稳定的Mergesort变体。尽管细节相当复杂，最好通过以下方式描述[男人自己](https://github.com/python/cpython/blob/master/Objects/listsort.txt)或[维基百科页面](https://en.wikipedia.org/wiki/Timsort)，基础知识易于理解。虽然 Mergesort 通常以递归方式工作，但 Timsort 以迭代方式工作。它从左到右处理数组，并查找所谓的*运行*.运行只是一个已排序的序列。这包括“以错误的方式”排序的序列，因为这些序列可以简单地反转以形成运行。在分拣过程开始时，根据输入的长度确定最小运行长度。如果 Timsort 找不到此最小运行长度的自然运行，则使用插入排序“人为地提升”运行。

以这种方式找到的运行使用堆栈进行跟踪，该堆栈会记住起始索引和每次运行的长度。堆栈上的运行时会不时合并在一起，直到只剩下一个已排序的运行。Timsort在决定合并哪些运行时试图保持平衡。一方面，您希望尽早尝试合并，因为这些运行的数据很有可能已经在缓存中，另一方面，您希望尽可能晚地合并以利用可能出现的数据中的模式。为了实现这一点，Timsort维护了两个不变量。若`A`,`B`和`C`是三个最顶层的运行：

*   `|C| > |B| + |A|`
*   `|B| > |A|`

![Runs stack before and after merging A with B](../_img/array-sort/runs-stack.svg)

该图显示了以下情况：`|A| > |B|`所以`B`与两个游程中较小的一个合并。

请注意，Timsort 仅合并连续的运行，这是保持稳定性所必需的，否则将在运行之间传输相等的元素。此外，第一个不变量确保运行长度的增长速度至少与斐波那契数列一样快，当我们知道最大数组长度时，运行堆栈的大小有一个上限。

现在可以看到，已经排序的序列在 O（n） 中排序，因为这样的数组将导致不需要合并的单个运行。最坏的情况是O（n log n）。这些算法属性以及Timsort的稳定性是我们最终选择Timsort而不是Quicksort的几个原因。

### 在扭矩中实现 Timsort

内置通常具有不同的代码路径，这些代码路径是在运行时根据各种变量选择的。最通用的版本可以处理任何类型的对象，无论它是否是`JSProxy`，具有拦截器，或者在检索或设置属性时需要执行原型链查找。
在大多数情况下，通用路径相当慢，因为它需要考虑所有可能性。但是，如果我们预先知道要排序的对象很简单`JSArray`只包含Smis，所有这些昂贵`[[Get]]`和`[[Set]]`操作可以替换为简单的加载和存储到`FixedArray`.主要区别在于[`ElementsKind`](/blog/elements-kinds).

现在的问题变成了如何实现快速路径。核心算法对所有算法都保持不变，但基于`ElementsKind`.我们可以实现此目的的一种方法是在每个呼叫站点上调度到正确的“访问器”。想象一下，每个“加载”/“存储”操作都有一个开关，我们根据选择的快速路径选择不同的分支。

另一种解决方案（这是尝试的第一种方法）是为每个快速路径复制整个内置内容一次，并内联正确的加载/存储访问方法。这种方法对于Timsort来说是不可行的，因为它是一个很大的内置，并且为每个快速路径制作一个副本总共需要106 KB，这对于单个内置来说太多了。

最终的解决方案略有不同。每个快速路径的每个加载/存储操作都放入其自己的“迷你内置”中。请参阅代码示例，其中显示了`FixedDoubleArray`s.

```torque
Load<FastDoubleElements>(
    context: Context, sortState: FixedArray, elements: HeapObject,
    index: Smi): Object {
  try {
    const elems: FixedDoubleArray = UnsafeCast<FixedDoubleArray>(elements);
    const value: float64 =
        LoadDoubleWithHoleCheck(elems, index) otherwise Bailout;
    return AllocateHeapNumberWithValue(value);
  }
  label Bailout {
    // The pre-processing step removed all holes by compacting all elements
    // at the start of the array. Finding a hole means the cmp function or
    // ToString changes the array.
    return Failure(sortState);
  }
}
```

相比之下，最通用的“加载”操作只是调用`GetProperty`.但是，虽然上面的版本生成了高效，快速的机器代码来加载和转换`Number`,`GetProperty`是对另一个内置的调用，该内置可能涉及原型链查找或调用访问器函数。

```js
builtin Load<ElementsAccessor : type>(
    context: Context, sortState: FixedArray, elements: HeapObject,
    index: Smi): Object {
  return GetProperty(context, elements, index);
}
```

然后，快速路径只是变成一组函数指针。这意味着我们只需要核心算法的一个副本，同时预先设置所有相关函数指针一次。虽然这大大减少了所需的代码空间（低至20k），但它的代价是每个访问站点都有一个间接分支。最近对使用的更改甚至加剧了这种情况[嵌入式内置](/blog/embedded-builtins).

### 排序状态

![](../_img/array-sort/sort-state.svg)

上图显示了“排序状态”。这是一个`FixedArray`在排序时跟踪所有需要的东西。每次`Array#sort`被调用，则分配这样的排序状态。条目 4 到 7 是上面讨论的包含快速路径的函数指针集。

每次我们从用户JavaScript代码返回时，都会使用“check”内置功能，以检查我们是否可以继续当前的快速路径。为此，它使用“初始接收器映射”和“初始接收器长度”。 如果用户代码修改了当前对象，我们只需放弃排序运行，将所有指针重置为其最通用的版本，然后重新启动排序过程。插槽 8 中的“救助状态”用于表示此重置。

“比较”条目可以指向两个不同的内置项。一个调用用户提供的比较函数，而另一个实现默认比较，该比较调用`toString`在这两个论点上，然后进行词典比较。

其余字段（快速路径 ID 除外）是特定于 Timsort 的。运行堆栈（如上所述）初始化为 85 的大小，这足以对长度为 2 的数组进行排序<sup>64</sup>.临时数组用于合并运行。它根据需要增加尺寸，但永远不会超过`n/2`哪里`n`是输入长度。

### 性能权衡

将排序从自托管 JavaScript 转移到 Torque 需要权衡性能。如`Array#sort`是用Torque编写的，它现在是一个静态编译的代码段，这意味着我们仍然可以为某些代码构建快速路径。[`ElementsKind`s](/blog/elements-kinds)但它永远不会像高度优化的TurboFan版本那样快，可以利用类型反馈。另一方面，如果代码不够热，无法保证JIT编译或调用站点是巨型的，我们就会遇到解释器或慢速/通用版本。自托管JavaScript版本的解析，编译和可能的优化也是Torque实现不需要的开销。

虽然扭矩方法不会为排序带来相同的峰值性能，但它确实避免了性能悬崖。结果是排序性能比以前更可预测。请记住，Torque非常不稳定，除了针对CSA之外，它将来可能会针对TurboFan，从而允许JIT编译用Torque编写的代码。

### 微基准标记

在我们开始之前`Array#sort`，我们添加了许多不同的微观基准，以更好地了解重新实现将产生的影响。第一个图表显示了使用用户提供的比较功能对各种元素排序的“正常”用例。

请记住，在这些情况下，JIT编译器可以做很多工作，因为排序几乎是我们所做的一切。这也允许优化编译器在JavaScript版本中内联比较函数，而在Torque情况下，我们有从内置到JavaScript的调用开销。尽管如此，我们在几乎所有情况下的表现都更好。

![](../_img/array-sort/micro-bench-basic.svg)

下一个图表显示了在处理已完全排序的数组或具有已单向排序的子序列时 Timsort 的影响。该图表使用快速排序作为基线，并显示Timsort的加速（在“DownDown”的情况下，最多17×数组由两个反向排序的序列组成）。可以看出，除了随机数据的情况外，Timsort在所有其他情况下的表现都更好，即使我们正在排序`PACKED_SMI_ELEMENTS`，其中Quicksort在上面的微板架标记中优于Timsort。

![](../_img/array-sort/micro-bench-presorted.svg)

### 网络工具基准测试

这[网络工具基准测试](https://github.com/v8/web-tooling-benchmark)是Web开发人员通常使用的工具的工作负载的集合，例如Babel和TypeScript。该图表使用 JavaScript Quicksort 作为基线，并将 Timsort 的加速与它进行比较。在几乎所有的基准测试中，除了chai之外，我们都保持相同的性能。

![](../_img/array-sort/web-tooling-benchmark.svg)

柴基准的支出*第三个*在单个比较函数（字符串距离计算）中的时间。基准测试是chai本身的测试套件。由于数据的原因，在这种情况下，Timsort需要更多的比较，这对整体运行时有更大的影响，因为如此大一部分时间都花在该特定的比较函数中。

### 对内存的影响

在浏览大约 50 个站点（在移动设备和桌面设备上）时分析 V8 堆快照时，没有显示任何内存回归或改进。一方面，这是令人惊讶的：从Quicksort到Timsort的转换引入了对用于合并运行的临时数组的需求，该数组可能比用于采样的临时数组大得多。另一方面，这些临时阵列的生存期非常短（仅在`sort`call），并且可以在 V8 的新空间中相当快地分配和丢弃。

## 结论

总而言之，我们对在 Torque 中实现的 Timsort 的算法属性和可预测的性能行为感觉要好得多。Timsort 从 V8 v7.0 和 Chrome 70 开始提供。祝您排序愉快！
