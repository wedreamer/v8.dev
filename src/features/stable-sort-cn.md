***

标题： '稳定`Array.prototype.sort`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-07-02
    标签：
*   ECMAScript
*   ES2019
*   io19
    描述： 'Array.prototype.sort 现在保证是稳定的。
    推文：“1146067251302244353”

***

假设你有一组狗，每只狗都有一个名字和一个评级。（如果这听起来像一个奇怪的例子，你应该知道有一个Twitter帐户专门研究这个......不要问！

```js
// Note how the array is pre-sorted alphabetically by `name`.
const doggos = [
  { name: 'Abby',   rating: 12 },
  { name: 'Bandit', rating: 13 },
  { name: 'Choco',  rating: 14 },
  { name: 'Daisy',  rating: 12 },
  { name: 'Elmo',   rating: 12 },
  { name: 'Falco',  rating: 13 },
  { name: 'Ghost',  rating: 14 },
];
// Sort the dogs by `rating` in descending order.
// (This updates `doggos` in place.)
doggos.sort((a, b) => b.rating - a.rating);
```

该数组按名称的字母顺序预先排序。要按评级排序（因此我们首先获得评分最高的狗），我们使用`Array#sort`，传入比较评级的自定义回调。这是您可能期望的结果：

```js
[
  { name: 'Choco',  rating: 14 },
  { name: 'Ghost',  rating: 14 },
  { name: 'Bandit', rating: 13 },
  { name: 'Falco',  rating: 13 },
  { name: 'Abby',   rating: 12 },
  { name: 'Daisy',  rating: 12 },
  { name: 'Elmo',   rating: 12 },
]
```

狗按评级排序，但在每个评级中，它们仍然按名称的字母顺序排序。例如，Choco 和 Ghost 具有相同的 14 分，但 Choco 在排序结果中出现在 Ghost 之前，因为这也是它们在原始数组中的顺序。

然而，要得到这个结果，JavaScript引擎不能只使用*任何*排序算法 — 它必须是所谓的“稳定排序”。很长一段时间，JavaScript规范不需要排序稳定性`Array#sort`，而是将其留给实现。由于此行为未指定，因此您也可能获得此排序结果，其中Ghost现在突然出现在Choco之前：

```js
[
  { name: 'Ghost',  rating: 14 }, // 😢
  { name: 'Choco',  rating: 14 }, // 😢
  { name: 'Bandit', rating: 13 },
  { name: 'Falco',  rating: 13 },
  { name: 'Abby',   rating: 12 },
  { name: 'Daisy',  rating: 12 },
  { name: 'Elmo',   rating: 12 },
]
```

换句话说，JavaScript开发人员不能依赖排序稳定性。在实践中，这种情况更加令人愤怒，因为一些JavaScript引擎会对短数组使用稳定的排序，而对较大的数组使用不稳定的排序。这真的很令人困惑，因为开发人员会测试他们的代码，看到稳定的结果，但是当数组稍大时，突然在生产中得到一个不稳定的结果。

但有一些好消息。我们[建议更改规范](https://github.com/tc39/ecma262/pull/1340)这使得`Array#sort`稳定，并被接受。所有主要的JavaScript引擎现在都实现了一个稳定的`Array#sort`.作为JavaScript开发人员，这少了一件需要担心的事情。好！

（哦，还有[我们做了同样的事情`TypedArray`s](https://github.com/tc39/ecma262/pull/1433)：该排序现在也很稳定。

：：：备注
**注意：**虽然现在每个规范都需要稳定性，但JavaScript引擎仍然可以自由地实现他们喜欢的任何排序算法。[V8 使用 Timsort](/blog/array-sort#timsort)例如。该规范不要求任何特定的排序算法。
:::

## 功能支持 { #support }

### 稳定`Array.prototype.sort`{ #support-stable-array-sort }

<feature-support chrome="70 /blog/v8-release-70#javascript-language-features"
              firefox="yes"
              safari="yes"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-array"></feature-support>

### 稳定`%TypedArray%.prototype.sort`{ #support-stable-typedarray-sort }

<feature-support chrome="74 https://bugs.chromium.org/p/v8/issues/detail?id=8567"
              firefox="67 https://bugzilla.mozilla.org/show_bug.cgi?id=1290554"
              safari="yes"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-typed-arrays"></feature-support>
