***

标题： '<i>快速特性</i>在 V8' 中
作者： '卡米洛布鲁尼 （[@camillobruni](https://twitter.com/camillobruni)），也是[“快速`for`-`in`"](/blog/fast-for-in)'
化身：

*   '卡米洛-布鲁尼'
    日期： 2017-08-30 13：33：37
    标签：
*   内部
    描述： “这个技术深入探讨了 V8 如何在幕后处理 JavaScript 属性。

***

在这篇博客文章中，我们想解释一下 V8 如何在内部处理 JavaScript 属性。从JavaScript的角度来看，属性只需要几个区别。JavaScript对象的行为大多像字典，字符串键和任意对象作为值。但是，该规范确实以不同的方式处理整数索引属性和其他属性。[在迭代期间](https://tc39.es/ecma262/#sec-ordinaryownpropertykeys).除此之外，不同的属性的行为大致相同，无论它们是否是整数索引。

但是，出于性能和内存原因，在引擎盖下 V8 确实依赖于几种不同的属性表示形式。在这篇博客文章中，我们将解释 V8 如何在处理动态添加的属性时提供快速的属性访问。了解属性的工作原理对于解释优化方式至关重要，例如[内联缓存](http://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)在 V8 中工作。

本文介绍了处理整数索引属性和命名属性的区别。之后，我们将展示 V8 在添加命名属性时如何维护 HiddenClasses，以便提供一种快速识别对象形状的方法。然后，我们将继续深入了解如何根据使用情况优化命名属性以进行快速访问或快速修改。在最后一节中，我们将提供有关 V8 如何处理整数索引属性或数组索引的详细信息。

## 命名属性与元素

让我们从分析一个非常简单的对象开始，例如`{a: "foo", b: "bar"}`.此对象有两个命名属性，`"a"`和`"b"`.它没有任何属性名称的整数索引。数组索引属性（通常称为元素）在数组中最为突出。例如，数组`["foo", "bar"]`具有两个数组索引属性：0，值为“foo”，1，值为“bar”。这是 V8 一般如何处理属性的第一个主要区别。

下图显示了基本 JavaScript 对象在内存中的外观。

![](/\_img/fast-properties/jsobject.png)

元素和属性存储在两个单独的数据结构中，这使得添加和访问属性或元素对于不同的使用模式更加高效。

元素主要用于各种[`Array.prototype`方法](https://tc39.es/ecma262/#sec-properties-of-the-array-prototype-object)如`pop`或`slice`.鉴于这些函数访问连续范围内的属性，V8 在内部也将它们表示为简单数组 — 大多数情况下。在本文的后面部分，我们将解释我们有时如何切换到基于稀疏字典的表示以节省内存。

命名属性以类似的方式存储在单独的数组中。但是，与元素不同，我们不能简单地使用键来推断它们在属性数组中的位置;我们需要一些额外的元数据。在 V8 中，每个 JavaScript 对象都有一个关联的 HiddenClass。HiddenClass 存储有关对象形状的信息，以及从属性名称到索引到属性的映射等信息。为了使事情复杂化，我们有时会使用字典作为属性，而不是简单的数组。我们将在专门的章节中更详细地解释这一点。

**本节要点：**

*   数组索引属性存储在单独的元素存储区中。
*   命名属性存储在属性存储区中。
*   元素和属性可以是数组或字典。
*   每个 JavaScript 对象都有一个关联的 HiddenClass，用于保存有关对象形状的信息。

## HiddenClasses and DescriptorArrays

在解释了元素和命名属性的一般区别之后，我们需要看看HiddenClasses在V8中是如何工作的。此 HiddenClass 存储有关对象的元信息，包括对象上的属性数和对对象原型的引用。HiddenClasses在概念上类似于典型的面向对象编程语言中的类。但是，在基于原型的语言（如JavaScript）中，通常不可能预先知道类。因此，在本例中，V8，HiddenClasses 是动态创建的，并随着对象的变化而动态更新。HiddenClasses作为对象形状的标识符，因此是V8优化编译器和内联缓存的非常重要的成分。例如，如果优化编译器可以通过HiddenClass确保对象结构兼容，则可以直接内联属性访问。

让我们来看看HiddenClass的重要部分。

![](/\_img/fast-properties/hidden-class.png)

在 V8 中，JavaScript 对象的第一个字段指向 HiddenClass。（实际上，对于 V8 堆上并由垃圾回收器管理的任何对象，都是如此。就属性而言，最重要的信息是第三位字段，它存储属性的数量，以及指向描述符数组的指针。描述符数组包含有关命名属性的信息，如名称本身和存储值的位置。请注意，我们在这里不跟踪整数索引属性，因此描述符数组中没有条目。

关于HiddenClasses的基本假设是具有相同结构的对象 - 例如，以相同顺序相同的命名属性 - 共享相同的HiddenClass。为了实现这一点，当一个属性被添加到一个对象时，我们使用一个不同的HiddenClass。在下面的示例中，我们从一个空对象开始，并添加三个命名属性。

![](/\_img/fast-properties/adding-properties.png)

每次添加新属性时，对象的 HiddenClass 都会更改。在后台，V8 创建了一个将 HiddenClasses 链接在一起的过渡树。V8 知道在将属性 “a” 添加到空对象时要采用哪个 HiddenClass。此过渡树可确保在以相同的顺序添加相同的属性时，最终会得到相同的最终 HiddenClass。下面的示例显示，即使我们在两者之间添加简单的索引属性，我们也会遵循相同的过渡树。

![](/\_img/fast-properties/transitions.png)

但是，如果我们创建一个新对象，该对象添加了不同的属性，则在本例中为属性`"d"`，V8 为新的 HiddenClasses 创建了一个单独的分支。

![](/\_img/fast-properties/transition-trees.png)

**本节要点：**

*   具有相同结构的对象（相同顺序的相同属性）具有相同的 HiddenClass
*   默认情况下，添加的每个新命名属性都会导致创建新的 HiddenClass。
*   添加数组索引属性不会创建新的隐藏类。

## 三种不同类型的命名属性

在概述了 V8 如何使用 HiddenClasses 来跟踪对象的形状之后，让我们深入了解这些属性的实际存储方式。如上面的介绍中所述，有两种基本类型的属性：命名属性和索引属性。以下部分介绍命名属性。

一个简单的对象，例如`{a: 1, b: 2}`可以在 V8 中具有各种内部表示形式。虽然JavaScript对象的行为或多或少像外部的简单字典，但V8试图避免字典，因为它们阻碍了某些优化，例如[内联缓存](https://en.wikipedia.org/wiki/Inline_caching)我们将在单独的帖子中解释。

**对象内属性与常规属性：**V8 支持所谓的对象内属性，这些属性直接存储在对象本身上。这些是 V8 中可用的最快属性，因为它们无需任何间接寻址即可访问。对象内属性的数量由对象的初始大小预先确定。如果添加的属性多于对象中的空间，则这些属性将存储在属性存储区中。属性存储添加一个间接层，但可以独立增长。

![](/\_img/fast-properties/in-object-properties.png)

**快速与慢速属性：**下一个重要的区别是快速属性和慢速属性之间的区别。通常，我们将存储在线性属性存储中的属性定义为“快速”。快速属性只需通过属性存储中的索引即可访问。要从属性的名称获取属性存储区中的实际位置，我们必须查阅 HiddenClass 上的描述符数组，如前所述。

![](/\_img/fast-properties/fast-vs-slow-properties.png)

但是，如果在对象中添加和删除了许多属性，则可能会产生大量的时间和内存开销来维护描述符数组和HiddenClasses。因此，V8 还支持所谓的慢属性。具有慢速属性的对象具有自包含的字典作为属性存储。所有属性元信息不再存储在 HiddenClass 上的描述符数组中，而是直接存储在属性字典中。因此，可以在不更新 HiddenClass 的情况下添加和删除属性。由于内联缓存不能与字典属性一起使用，因此后者通常比快速属性慢。

**本节要点：**

*   有三种不同的命名属性类型：对象内、快速和慢速/字典。
    1.  对象内属性直接存储在对象本身上，并提供最快的访问。
    2.  快速属性存在于属性存储中，所有元信息都存储在HiddenClass上的描述符数组中。
    3.  慢速属性存在于自包含的属性字典中，元信息不再通过HiddenClass共享。
*   慢速属性允许有效地删除和添加属性，但访问速度比其他两种类型慢。

## 元素或数组索引属性

到目前为止，我们已经研究了命名属性，并忽略了数组中常用的整数索引属性。整数索引属性的处理并不比命名属性复杂。即使所有索引属性始终单独保存在元素存储中，也有[20](https://cs.chromium.org/chromium/src/v8/src/elements-kind.h?q=elements-kind.h\&sq=package:chromium\&dr\&l=14)不同类型的元素！

**填充或多孔元素：**V8的第一个主要区别是元素后备存储是否包装或其中有孔。如果删除索引元素，或者例如，未定义它，则会在后备存储中出现漏洞。一个简单的例子是`[1,,3]`其中第二个条目是一个孔。下面的示例阐释了此问题：

```js
const o = ['a', 'b', 'c'];
console.log(o[1]);          // Prints 'b'.

delete o[1];                // Introduces a hole in the elements store.
console.log(o[1]);          // Prints 'undefined'; property 1 does not exist.
o.__proto__ = {1: 'B'};     // Define property 1 on the prototype.

console.log(o[0]);          // Prints 'a'.
console.log(o[1]);          // Prints 'B'.
console.log(o[2]);          // Prints 'c'.
console.log(o[3]);          // Prints undefined
```

![](/\_img/fast-properties/hole.png)

简而言之，如果接收器上不存在属性，我们必须继续查看原型链。假设元素是自包含的，例如，我们不在HiddenClass上存储有关当前索引属性的信息，我们需要一个名为the_hole的特殊值来标记不存在的属性。这对于数组函数的性能至关重要。如果我们知道没有孔，即元素存储是打包的，我们可以执行本地操作，而无需在原型链上进行昂贵的查找。

**快速或字典元素：**对元素的第二个主要区别是它们是快速还是字典模式。快速元素是简单的 VM 内部数组，其中属性索引映射到元素存储中的索引。但是，对于只有很少条目占用的非常大的稀疏/多孔数组，这种简单的表示是相当浪费的。在本例中，我们使用基于字典的表示来节省内存，但代价是访问速度稍慢：

```js
const sparseArray = [];
sparseArray[9999] = 'foo'; // Creates an array with dictionary elements.
```

在此示例中，分配包含 10k 个条目的完整数组将相当浪费。相反，V8 创建了一个字典，我们在其中存储了一个键值描述符三元组。在这种情况下，关键是`'9999'`和值`'foo'`并使用默认描述符。鉴于我们没有办法在HiddenClass上存储描述符详细信息，因此每当您使用自定义描述符定义索引属性时，V8都会求助于慢速元素：

```js
const array = [];
Object.defineProperty(array, 0, {value: 'fixed' configurable: false});
console.log(array[0]);      // Prints 'fixed'.
array[0] = 'other value';   // Cannot override index 0.
console.log(array[0]);      // Still prints 'fixed'.
```

在此示例中，我们在数组上添加了一个不可配置的属性。此信息存储在慢速元素字典三元组的描述符部分中。重要的是要注意，Array函数在具有慢速元素的对象上执行速度要慢得多。

**Smi和双元素：**对于快速元素，V8中还有另一个重要的区别。例如，如果您只将整数存储在数组中（一种常见的用例），则GC不必查看数组，因为整数直接编码为所谓的小整数（Smis）。另一种特殊情况是仅包含双精度值的数组。与 Smis 不同，浮点数通常表示为占据多个单词的完整对象。但是，V8 为纯双精度数组存储原始双精度，以避免内存和性能开销。以下示例列出了 Smi 和双元素的 4 个示例：

```js
const a1 = [1,   2, 3];  // Smi Packed
const a2 = [1,    , 3];  // Smi Holey, a2[1] reads from the prototype
const b1 = [1.1, 2, 3];  // Double Packed
const b2 = [1.1,  , 3];  // Double Holey, b2[1] reads from the prototype
```

**特殊元素：**到目前为止，我们根据这些信息涵盖了20种不同元素中的7种。为简单起见，我们为TypedArrays排除了9种元素类型，为字符串包装器排除了另外两种元素类型，最后但并非最不重要的是，为参数对象排除了两种特殊元素类型。

**元素访问器：**您可以想象，我们并不完全热衷于在C++中编写数组函数20次，每次一次[元素种类](/blog/elements-kinds).这就是一些C++魔力发挥作用的地方。我们没有一遍又一遍地实现 Array 函数，而是构建了`ElementsAccessor`在这里，我们主要只需要实现从后备存储访问元素的简单函数。这`ElementsAccessor`依赖[断续器](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)以创建每个 Array 函数的专用版本。所以如果你称之为类似的东西`slice`在数组上，V8 在内部调用用 C++ 编写的内置，并通过`ElementsAccessor`到函数的专用版本：

![](/\_img/fast-properties/elements-accessor.png)

**本节要点：**

*   有快速和字典模式索引的属性和元素。
*   快速属性可以打包，也可以包含指示索引属性已被删除的孔。
*   元素专门用于其内容，以加快数组功能并减少GC开销。

了解属性的工作原理是 V8 中许多优化的关键。对于JavaScript开发人员来说，许多这些内部决策并不直接可见，但它们解释了为什么某些代码模式比其他代码模式更快。更改属性或元素类型通常会导致 V8 创建不同的 HiddenClass，这可能导致类型污染[阻止 V8 生成最佳代码](http://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html).请继续关注有关 V8 的 VM 内部如何工作的更多文章。
