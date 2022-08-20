***

标题：“优化哈希表：隐藏哈希代码”
作者： '[萨提亚·古纳塞卡兰](https://twitter.com/\_gsathya)，哈希代码的守护者
化身：

*   'sathya-gunasekaran'
    发布日期：2018-01-29 13：33：37
    标签：
*   内部
    推文：“958046113390411776”
    描述： '一些JavaScript数据结构，如Map，Set，WeakSet和WeakMap，在引擎盖下使用哈希表。本文介绍了 V8 v6.3 如何提高哈希表的性能。

***

ECMAScript 2015 引入了几个新的数据结构，如 Map、Set、WeakSet 和 WeakMap，所有这些都在后台使用哈希表。这篇文章详细介绍了[最近的改进](https://bugs.chromium.org/p/v8/issues/detail?id=6404)如何[V8 v6.3+](/blog/v8-release-63)将密钥存储在哈希表中。

## 哈希代码

一个[*哈希函数*](https://en.wikipedia.org/wiki/Hash_function)用于将给定键映射到哈希表中的位置。一个*哈希代码*是在给定键上运行此哈希函数的结果。

在 V8 中，哈希代码只是一个随机数，与对象值无关。因此，我们不能重新计算它，这意味着我们必须存储它。

对于用作键的 JavaScript 对象，以前，哈希代码作为私有符号存储在对象上。V8 中的私有符号类似于[`Symbol`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)，除了它不是可枚举的，并且不会泄漏到用户空间JavaScript。

```js
function GetObjectHash(key) {
  let hash = key[hashCodeSymbol];
  if (IS_UNDEFINED(hash)) {
    hash = (MathRandom() * 0x40000000) | 0;
    if (hash === 0) hash = 1;
    key[hashCodeSymbol] = hash;
  }
  return hash;
}
```

这很有效，因为我们不必为哈希代码字段保留内存，直到将对象添加到哈希表，此时在对象上存储了新的私有符号。

V8还可以像使用IC系统的任何其他属性查找一样优化哈希代码符号查找，从而为哈希代码提供非常快速的查找。这很适用于[单态 IC 查找](https://en.wikipedia.org/wiki/Inline_caching#Monomorphic_inline_caching)，当键具有相同的[隐藏类](/).但是，大多数实际代码并不遵循此模式，并且键通常具有不同的隐藏类，从而导致速度变慢[巨态 IC 查找](https://en.wikipedia.org/wiki/Inline_caching#Megamorphic_inline_caching)的哈希代码。

私有符号方法的另一个问题是，它触发了[隐藏类转换](/#fast-property-access)在存储哈希代码的密钥中。这导致多态代码很差，不仅对于哈希代码查找，而且对于密钥和[去优化](https://floitsch.blogspot.com/2012/03/optimizing-for-v8-inlining.html)从优化的代码。

## JavaScript 对象支持存储

一个 JavaScript 对象 （`JSObject`） 在 V8 中使用两个词（除了它的标头）：一个词用于存储指向元素后备存储的指针，另一个词用于存储指向属性后备存储的指针。

元素后备存储用于存储看起来像[数组索引](https://tc39.es/ecma262/#sec-array-index)，而属性支持存储用于存储键为字符串或符号的属性。看到这个[V8 博客文章](/blog/fast-properties)由卡米洛布鲁尼有关这些后备商店的更多信息。

```js
const x = {};
x[1] = 'bar';      // ← stored in elements
x['foo'] = 'bar';  // ← stored in properties
```

## 隐藏哈希代码

存储哈希代码的最简单解决方案是将 JavaScript 对象的大小扩展一个字，并将哈希代码直接存储在对象上。但是，这会为未添加到哈希表的对象浪费内存。相反，我们可以尝试将哈希代码存储在元素存储或属性存储中。

元素后备存储是一个包含其长度和所有元素的数组。这里没有太多要做的，因为当我们不使用对象作为哈希表中的键时，将哈希码存储在保留的插槽（如第0个索引）中仍然会浪费内存。

让我们看一下属性后备存储。有两种数据结构用作属性支持存储：数组和字典。

与没有上限的元素后备存储中使用的数组不同，属性后备存储中使用的数组的上限为 1022 个值。出于性能原因，V8 在超过此限制时过渡到使用字典。（我稍微简化了这一点 - V8在其他情况下也可以使用字典，但是数组中可以存储的值的数量有一个固定的上限。

因此，属性后备存储有三种可能的状态：

1.  空（无属性）
2.  数组（最多可存储 1022 个值）
3.  字典

让我们逐一讨论。

### 属性后备存储为空

对于空情况，我们可以直接将哈希代码存储在此偏移量上`JSObject`.

![](/\_img/hash-code/properties-backing-store-empty.png)

### 属性后备存储是一个数组

V8 表示小于 2 的整数<sup>31</sup>（在 32 位系统上）未装箱，如[斯米](https://wingolog.org/archives/2011/05/18/value-representation-in-javascript-implementations)s.在 Smi 中，最低有效位是用于将其与指针区分开来的标记，而其余 31 位保存实际的整数值。

通常，数组将其长度存储为 Smi。由于我们知道此数组的最大容量仅为 1022，因此我们只需要 10 位来存储长度。我们可以使用剩余的21位来存储哈希代码！

![](/\_img/hash-code/properties-backing-store-array.png)

### 属性后备存储是字典

对于字典情况，我们将字典大小增加 1 个单词，以将哈希代码存储在字典开头的专用插槽中。在这种情况下，我们可能会浪费一字内存，因为大小的比例增加并不像数组情况那样大。

![](/\_img/hash-code/properties-backing-store-dictionary.png)

通过这些更改，哈希代码查找不再需要通过复杂的JavaScript属性查找机制。

## 性能改进

这[六速](https://github.com/kpdecker/six-speed)benchmark 跟踪 Map 和 Set 的性能，这些更改带来了大约 500% 的改进。

![](/\_img/hash-code/sixspeed.png)

此更改导致基本基准测试提高了 5%[阿瑞斯6](https://webkit.org/blog/7536/jsc-loves-es6/)也。

![](/\_img/hash-code/ares-6.png)

这也使[恩贝佩夫](http://emberperf.eviltrout.com/)测试Ember.js的基准套件。

![](/\_img/hash-code/emberperf.jpg)
