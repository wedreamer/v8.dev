***

标题： '快速`for`-`in`在 V8' 中
作者： '卡米洛布鲁尼 （[@camillobruni](http://twitter.com/camillobruni))'
化身：

*   '卡米洛-布鲁尼'
    日期： 2017-03-01 13：33：37
    标签：
*   内部
    描述： “这个技术深入探讨了 V8 是如何让 JavaScript 尽可能快地进入的。

***

`for`-`in`是许多框架中存在的广泛使用的语言功能。尽管它无处不在，但从实现的角度来看，它是更晦涩的语言结构之一。V8 竭尽全力使此功能尽可能快。在过去一年中，`for`-`in`完全符合规范，速度提高了 3 倍，具体取决于上下文。

许多流行的网站严重依赖for-in，并从其优化中受益。例如，在2016年初，Facebook在启动期间花费了大约7%的JavaScript时间来实施`for`-`in`本身。在维基百科上，这个数字甚至更高，约为8%。通过提高某些慢速机箱的性能，Chrome 51 显著提高了以下两个网站的性能：

![](/\_img/fast-for-in/wikipedia.png)

![](/\_img/fast-for-in/facebook.png)

维基百科和Facebook都将其总脚本时间提高了4%，原因如下：`for`-`in`改进。请注意，在同一时期，V8的其余部分也变得更快，这使得脚本编写的总体改进超过4%。

在本博客文章的其余部分，我们将解释我们如何设法加速此核心语言功能并同时修复长期存在的规范违规。

## 规格

***TL;DR;**由于性能原因，for-in 迭代语义是模糊的。*

当我们看到[规范文本`for`-`in`，它是以一种出乎意料的模糊方式写的](https://tc39.es/ecma262/#sec-for-in-and-for-of-statements)，这在不同的实现中是可观察的。让我们看一个迭代时的示例[代理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)设置了适当陷阱的对象。

```js
const proxy = new Proxy({ a: 1, b: 1},
  {
    getPrototypeOf(target) {
    console.log('getPrototypeOf');
    return null;
  },
  ownKeys(target) {
    console.log('ownKeys');
    return Reflect.ownKeys(target);
  },
  getOwnPropertyDescriptor(target, prop) {
    console.log('getOwnPropertyDescriptor name=' + prop);
    return Reflect.getOwnPropertyDescriptor(target, prop);
  }
});
```

在 V8/Chrome 56 中，您将获得以下输出：

    ownKeys
    getPrototypeOf
    getOwnPropertyDescriptor name=a
    a
    getOwnPropertyDescriptor name=b
    b

相反，在 Firefox 51 中，对于同一代码段，您将获得不同的语句顺序：

    ownKeys
    getOwnPropertyDescriptor name=a
    getOwnPropertyDescriptor name=b
    getPrototypeOf
    a
    b

两种浏览器都遵守规范，但有一次规范不强制执行明确的指令顺序。要正确理解这些漏洞，让我们看一下规范文本：

> 枚举对象属性 （ O ）
> 当使用参数 O 调用抽象操作 EnumerateObjectProperties 时，将执行以下步骤：
>
> 1.  断言：Type（O） 是 Object。
> 2.  返回一个迭代器对象 （25.1.1.2），其下一个方法循环访问 O 的可枚举属性的所有字符串值键。迭代器对象永远不会被 ECMAScript 代码直接访问。枚举属性的机制和顺序未指定，但必须符合下面指定的规则。

现在，通常规格说明在需要的确切步骤方面是精确的。但在这种情况下，他们指的是一个简单的散文列表，甚至执行的顺序也留给了实施者。通常，这样做的原因是规范的这些部分是在JavaScript引擎已经具有不同实现的事实之后编写的。规范试图通过提供以下说明来系紧松散的末端：

1.  迭代器的 throw 和 return 方法为 null，从不调用。
2.  迭代器的下一个方法处理对象属性，以确定是否应将属性键作为迭代器值返回。
3.  返回的属性键不包括符号键。
4.  在枚举过程中可能会删除目标对象的属性。
5.  在迭代器的下一个方法处理之前删除的属性将被忽略。如果在枚举期间向目标对象添加了新属性，则不保证在活动枚举中处理新添加的属性。
6.  在任何枚举中，迭代器的下一个方法最多返回一次属性名称。
7.  枚举目标对象的属性包括枚举其原型的属性，以及原型的原型，依此类推;但是，如果原型的属性与迭代器的下一个方法已处理的属性同名，则不会处理该属性。
8.  的价值观`[[Enumerable]]`在确定原型对象的属性是否已处理时，不考虑特性。
9.  原型对象的可枚举属性名称必须通过调用枚举对象属性来获取，该属性将原型对象作为参数传递。
10. 枚举对象属性必须通过调用其来获取目标对象自己的属性键`[[OwnPropertyKeys]]`内部方法。

这些步骤听起来很乏味，但是规范还包含一个显式且更具可读性的示例实现：

```js
function* EnumerateObjectProperties(obj) {
  const visited = new Set();
  for (const key of Reflect.ownKeys(obj)) {
    if (typeof key === 'symbol') continue;
    const desc = Reflect.getOwnPropertyDescriptor(obj, key);
    if (desc && !visited.has(key)) {
      visited.add(key);
      if (desc.enumerable) yield key;
    }
  }
  const proto = Reflect.getPrototypeOf(obj);
  if (proto === null) return;
  for (const protoKey of EnumerateObjectProperties(proto)) {
    if (!visited.has(protoKey)) yield protoKey;
  }
}
```

现在您已经做到了这一点，您可能已经从前面的示例中注意到 V8 并不完全遵循规范示例实现。首先，示例 for-in 生成器以增量方式工作，而 V8 则预先收集所有密钥 - 主要是出于性能原因。这完全没问题，事实上，规范文本明确指出未定义操作 A - J 的顺序。然而，正如您将在本文后面发现的那样，在一些角落的情况下，V8直到2016年才完全遵守规范。

## 枚举缓存

的示例实现`for`-`in`生成器遵循收集和生成密钥的增量模式。在 V8 中，属性键在第一步中收集，然后才在迭代阶段使用。对于 V8，这让一些事情变得更容易。要理解原因，我们需要看一下对象模型。

一个简单的对象，例如`{a:'value a', b:'value b', c:'value c'}`可以在 V8 中具有各种内部表示形式，我们将在有关属性的详细后续文章中展示。这意味着，根据我们拥有的属性类型（对象内、快速或慢速），实际属性名称存储在不同的地方。这使得收集可枚举密钥成为一项不平凡的任务。

V8 通过隐藏类或所谓的 Map 来跟踪对象的结构。具有相同映射的对象具有相同的结构。此外，每个 Map 都有一个共享数据结构，即描述符数组，其中包含有关每个属性的详细信息，例如属性在对象上的存储位置、属性名称以及可枚举性等详细信息。

让我们暂时假设我们的 JavaScript 对象已达到其最终形状，并且不会添加或删除更多属性。在这种情况下，我们可以使用描述符数组作为键的源。如果只有可枚举属性，则此方法有效。为了避免每次 V8 使用可通过 Map 的描述符数组访问的单独 EnumCache 时过滤掉不可枚举属性的开销。

![](/\_img/fast-for-in/enum-cache.png)

假设 V8 期望慢速字典对象频繁更改（即通过添加和删除属性），对于具有字典属性的慢速对象，没有描述符数组。因此，V8 不为慢速属性提供 EnumCache。类似的假设也适用于索引属性，因此它们也被排除在EnumCache之外。

让我们总结一下重要的事实：

*   贴图用于跟踪对象形状。
*   描述符数组存储有关属性的信息（名称、可配置性、可见性）。
*   描述符数组可以在地图之间共享。
*   每个描述符数组都可以有一个 EnumCache，仅列出可枚举的命名键，而不列出索引的属性名称。

## 的机制`for`-`in`

现在，您已经部分了解了 Maps 的工作原理以及 EnumCache 与描述符数组的关系。V8 通过字节码解释器 Ignition 和优化编译器 TurboFan 执行 JavaScript，两者都以类似的方式处理 for-in。为简单起见，我们将使用伪C++样式来解释如何在内部实现 for-in：

```js
// For-In Prepare:
FixedArray* keys = nullptr;
Map* original_map = object->map();
if (original_map->HasEnumCache()) {
  if (object->HasNoElements()) {
    keys = original_map->GetCachedEnumKeys();
  } else {
    keys = object->GetCachedEnumKeysWithElements();
  }
} else {
  keys = object->GetEnumKeys();
}

// For-In Body:
for (size_t i = 0; i < keys->length(); i++) {
  // For-In Next:
  String* key = keys[i];
  if (!object->HasProperty(key) continue;
  EVALUATE_FOR_IN_BODY();
}
```

For-in可以分为三个主要步骤：

1.  准备要迭代的密钥，
2.  获取下一个密钥，
3.  评估`for`-`in`身体。

“准备”步骤是这三个步骤中最复杂的，这是EnumCache发挥作用的地方。在上面的示例中，您可以看到 V8 直接使用 EnumCache，如果它存在，并且对象（及其原型）上没有元素（整数索引属性）。对于存在索引属性名称的情况，V8 将跳转到在 C++ 中实现的运行时函数，该函数将它们预先附加到现有的枚举缓存中，如以下示例所示：

```cpp
FixedArray* JSObject::GetCachedEnumKeysWithElements() {
  FixedArray* keys = object->map()->GetCachedEnumKeys();
  return object->GetElementsAccessor()->PrependElementIndices(object, keys);
}

FixedArray* Map::GetCachedEnumKeys() {
  // Get the enumerable property keys from a possibly shared enum cache
  FixedArray* keys_cache = descriptors()->enum_cache()->keys_cache();
  if (enum_length() == keys_cache->length()) return keys_cache;
  return keys_cache->CopyUpTo(enum_length());
}

FixedArray* FastElementsAccessor::PrependElementIndices(
      JSObject* object, FixedArray* property_keys) {
  Assert(object->HasFastElements());
  FixedArray* elements = object->elements();
  int nof_indices = CountElements(elements)
  FixedArray* result = FixedArray::Allocate(property_keys->length() + nof_indices);
  int insertion_index = 0;
  for (int i = 0; i < elements->length(); i++) {
    if (!HasElement(elements, i)) continue;
    result[insertion_index++] = String::FromInt(i);
  }
  // Insert property keys at the end.
  property_keys->CopyTo(result, nof_indices - 1);
  return result;
}
```

在没有找到现有EnumCache的情况下，我们再次跳转到C++并遵循最初提出的规范步骤：

```cpp
FixedArray* JSObject::GetEnumKeys() {
  // Get the receiver’s enum keys.
  FixedArray* keys = this->GetOwnEnumKeys();
  // Walk up the prototype chain.
  for (JSObject* object : GetPrototypeIterator()) {
     // Append non-duplicate keys to the list.
     keys = keys->UnionOfKeys(object->GetOwnEnumKeys());
  }
  return keys;
}

FixedArray* JSObject::GetOwnEnumKeys() {
  FixedArray* keys;
  if (this->HasEnumCache()) {
    keys = this->map()->GetCachedEnumKeys();
  } else {
    keys = this->GetEnumPropertyKeys();
  }
  if (this->HasFastProperties()) this->map()->FillEnumCache(keys);
  return object->GetElementsAccessor()->PrependElementIndices(object, keys);
}

FixedArray* FixedArray::UnionOfKeys(FixedArray* other) {
  int length = this->length();
  FixedArray* result = FixedArray::Allocate(length + other->length());
  this->CopyTo(result, 0);
  int insertion_index = length;
  for (int i = 0; i < other->length(); i++) {
    String* key = other->get(i);
    if (other->IndexOf(key) == -1) {
      result->set(insertion_index, key);
      insertion_index++;
    }
  }
  result->Shrink(insertion_index);
  return result;
}
```

这个简化的C++代码对应于V8中的实现，直到2016年初我们开始研究UnionOfKeys方法。如果你仔细观察，你会发现我们使用一种朴素的算法从列表中排除重复项，如果我们在原型链上有很多键，这可能会产生不良的性能。这就是我们决定在下一节中进行优化的方式。

## 问题`for`-`in`

正如我们在上一节中已经暗示的那样，UnionOfKeys 方法在最坏的情况下性能较差。它基于一个有效的假设，即大多数对象都具有快速属性，因此将从EnumCache中受益。第二个假设是，原型链上只有很少的可枚举属性限制了查找重复项所花费的时间。但是，如果对象在原型链上具有慢速字典属性和许多键，则UnionOfKeys将成为瓶颈，因为我们每次输入for-in时都必须收集可枚举属性名称。

除了性能问题之外，现有算法还有另一个问题，因为它不符合规范。V8 多年来一直错误地引用了以下示例：

```js
var o = {
  __proto__ : {b: 3},
  a: 1
};
Object.defineProperty(o, 'b', {});

for (var k in o) console.log(k);
```

输出：

    a
    b

也许违反直觉，这应该只是打印出来`a`而不是`a`和`b`.如果您回想起本文开头的规范文本，步骤 G 和 J 意味着原型链上接收器阴影属性上的不可枚举属性。

为了使事情变得更加复杂，ES6引入了[代理](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Proxy)对象。这打破了V8代码的许多假设。要以符合规范的方式实现 for-in，我们必须触发总共 13 个不同代理陷阱中的以下 5 个。

：：：表包装器
|内部方法|处理程序方法|
|--------------------- |-------------------------- |
|`[[GetPrototypeOf]]`|`getPrototypeOf`|
|`[[GetOwnProperty]]`|`getOwnPropertyDescriptor`|
|`[[HasProperty]]`|`has`|
|`[[Get]]`|`get`|
|`[[OwnPropertyKeys]]`|`ownKeys`|
:::

这需要原始GetEnumKeys代码的重复版本，该代码试图更紧密地遵循规范示例实现。ES6 代理和缺乏处理阴影属性是我们重构如何在 2016 年初提取所有密钥以供输入的核心动力。

## 这`KeyAccumulator`

我们引入了一个单独的帮助程序类，`KeyAccumulator`，它处理了收集钥匙的复杂性`for`-`in`.随着ES6规范的增长，新功能如`Object.keys`或`Reflect.ownKeys`需要自己稍微修改一下收集钥匙的版本。通过拥有一个可配置的位置，我们可以提高`for`-`in`并避免重复的代码。

这`KeyAccumulator`由一个快速部分组成，该部分仅支持一组有限的操作，但能够非常有效地完成它们。慢速累加器支持所有复杂情况，如ES6代理。

![](/\_img/fast-for-in/keyaccumulator.png)

为了正确过滤掉阴影属性，我们必须维护一个单独的不可枚举属性列表，到目前为止我们已经看到。出于性能原因，我们只有在确定对象的原型链上存在可枚举属性后才会执行此操作。

## 性能改进

随着`KeyAccumulator`到位后，又有一些模式变得可行以进行优化。第一个是避免原始UnionOfKeys方法的嵌套循环，这会导致缓慢的角落情况。在第二步中，我们执行了更详细的预检查，以利用现有的EnumCaches并避免不必要的复制步骤。

为了说明符合规范的实现速度更快，让我们看一下以下四个不同的对象：

```js
var fastProperties = {
  __proto__ : null,
  'property 1': 1,
  …
  'property 10': n
};

var fastPropertiesWithPrototype = {
  'property 1': 1,
  …
  'property 10': n
};

var slowProperties = {
  __proto__ : null,
  'dummy': null,
  'property 1': 1,
  …
  'property 10': n
};
delete slowProperties['dummy']

var elements = {
  __proto__: null,
  '1': 1,
  …
  '10': n
}
```

*   这`fastProperties`对象具有标准的快速属性。
*   这`fastPropertiesWithPrototype`对象在原型链上具有其他不可枚举的属性，方法是使用`Object.prototype`.
*   这`slowProperties`对象具有慢速字典属性。
*   这`elements`对象只有索引属性。

下图比较了运行`for`-`in`在没有我们的优化编译器帮助的情况下，在紧密循环中循环一百万次。

![](/\_img/fast-for-in/keyaccumulator-benchmark.png)

正如我们在引言中概述的那样，这些改进在维基百科和Facebook上变得非常明显。

![](/\_img/fast-for-in/wikipedia.png)

![](/\_img/fast-for-in/facebook.png)

除了Chrome 51中可用的初始改进之外，第二次性能调整还产生了另一个重大改进。下图显示了我们在 Facebook 页面上启动期间在脚本编写过程中花费的总时间的跟踪数据。围绕 V8 修订版 37937 的选定范围相当于额外提高了 4% 的性能！

![](/\_img/fast-for-in/fastkeyaccumulator.png)

强调改进的重要性`for`-`in`我们可以依靠我们在2016年构建的工具中的数据，该工具允许我们在一组网站上提取V8测量值。下表显示了在 V8 C++ Chrome 49 的入口点（运行时函数和内置）中花费的相对时间，这些时间大约是[25个具有代表性的现实世界网站](/blog/real-world-performance).

：：：表包装器
|职位|名称|总时间|
|:------: |------------------------------------- |---------- |
|1 |`CreateObjectLiteral`|1.10% |
|2 |`NewObject`|0.90% |
|3 |`KeyedGetProperty`|0.70% |
|4 |`GetProperty`|0.60% |
|5 |`ForInEnumerate`|0.60% |
|6 |`SetProperty`|0.50% |
|7 |`StringReplaceGlobalRegExpWithString`|0.30% |
|8 |`HandleApiCallConstruct`|0.30% |
|9 |`RegExpExec`|0.30% |
|10 |`ObjectProtoToString`|0.30% |
|11 |`ArrayPush`|0.20% |
|12 |`NewClosure`|0.20% |
|13 |`NewClosure_Tenured`|0.20% |
|14 |`ObjectDefineProperty`|0.20% |
|15 |`HasProperty`|0.20% |
|16 |`StringSplit`|0.20% |
|17 |`ForInFilter`|0.10% |
:::

最重要的`for`-`in`助手分别位于第 5 和第 17 位，平均占在网站上编写脚本所花费总时间的 0.7%。在 Chrome 57 中`ForInEnumerate`已降至总时间的 0.2%，并且`ForInFilter`由于在汇编程序中写入的快速路径，因此低于测量阈值。
