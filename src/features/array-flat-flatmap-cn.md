***

标题： '`Array.prototype.flat`和`Array.prototype.flatMap`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-06-11
    标签：
*   ECMAScript
*   ES2019
*   io19
    描述： 'Array.prototype.flat 将数组展平到指定的深度。Array.prototype.flatMap相当于做一个地图，然后单独做一个平面图。
    推文：“1138457106380709891”

***

## `Array.prototype.flat`{ #flat }

此示例中的数组是几个级别的深度：它包含一个数组，而数组又包含另一个数组。

```js
const array = [1, [2, [3]]];
//            ^^^^^^^^^^^^^ outer array
//                ^^^^^^^^  inner array
//                    ^^^   innermost array
```

`Array#flat`返回给定数组的平展版本。

```js
array.flat();
// → [1, 2, [3]]

// …is equivalent to:
array.flat(1);
// → [1, 2, [3]]
```

默认深度为`1`，但您可以传递任何数字以递归方式平展到该深度。为了保持递归平展，直到结果不再包含嵌套数组，我们传递`Infinity`.

```js
// Flatten recursively until the array contains no more nested arrays:
array.flat(Infinity);
// → [1, 2, 3]
```

为什么此方法称为`Array.prototype.flat`而不是`Array.prototype.flatten`?[阅读我们的#SmooshGate文章找出答案！](https://developers.google.com/web/updates/2018/03/smooshgate)

## `Array.prototype.flatMap`{ #flatMap }

这是另一个示例。我们有一个`duplicate`函数，该函数采用一个值，并返回一个包含该值两次的数组。如果我们申请`duplicate`对于数组中的每个值，我们最终得到一个嵌套数组。

```js
const duplicate = (x) => [x, x];

[2, 3, 4].map(duplicate);
// → [[2, 2], [3, 3], [4, 4]]
```

然后，您可以致电`flat`以平展数组：

```js
[2, 3, 4].map(duplicate).flat(); // 🐌
// → [2, 2, 3, 3, 4, 4]
```

由于这种模式在函数式编程中非常普遍，现在有一个专用的`flatMap`方法为它。

```js
[2, 3, 4].flatMap(duplicate); // 🚀
// → [2, 2, 3, 3, 4, 4]
```

`flatMap`比做一个更有效率`map`后跟`flat`分别。

对 以下用例感兴趣`flatMap`?退房[阿克塞尔·劳施迈尔的解释](https://exploringjs.com/impatient-js/ch_arrays.html#flatmap-mapping-to-zero-or-more-values).

## `Array#{flat,flatMap}`支持 { #support }

<feature-support chrome="69 /blog/v8-release-69#javascript-language-features"
              firefox="62"
              safari="12"
              nodejs="11"
              babel="yes https://github.com/zloirock/core-js#ecmascript-array"></feature-support>
