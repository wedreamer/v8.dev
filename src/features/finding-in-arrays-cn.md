***

标题： '查找元素`Array`s 和 TypedArrays'
作者： '郭淑宇 （[@\_shu](https://twitter.com/\_shu))'
化身：

*   “舒玉国”
    日期： 2021-10-27
    标签：
*   ECMAScript
    描述： '在数组和 Typed 数组中查找元素的 JavaScript 方法'
    推文：“1453354998063149066”

***

## 从头开始查找元素

查找满足某个条件的元素`Array`是一项常见任务，并且通过`find`和`findIndex`方法`Array.prototype`和各种TypedArray原型。`Array.prototype.find`取谓词并返回数组中该谓词返回的第一个元素`true`.如果谓词不返回`true`对于任何元素，该方法返回`undefined`.

```js
const inputArray = [{v:1}, {v:2}, {v:3}, {v:4}, {v:5}];
inputArray.find((element) => element.v % 2 === 0);
// → {v:2}
inputArray.find((element) => element.v % 7 === 0);
// → undefined
```

`Array.prototype.findIndex`工作方式类似，只是它在找到时返回索引，并且`-1`未找到时。TypedArray 版本的`find`和`findIndex`工作原理完全相同，唯一的区别是它们在TypedArray实例而不是Array实例上运行。

```js
inputArray.findIndex((element) => element.v % 2 === 0);
// → 1
inputArray.findIndex((element) => element.v % 7 === 0);
// → -1
```

## 从末尾查找元素

如果要查找`Array`?这种用例通常会自然而然地出现，例如选择对多个匹配项进行重复数据删除以支持最后一个元素，或者提前知道该元素可能接近`Array`.随着`find`方法，一个解决方案是首先反转输入，如下所示：

```js
inputArray.reverse().find(predicate)
```

但是，这反转了原始版本`inputArray`就地，这有时是不可取的。

随着`findLast`和`findLastIndex`方法，这个用例可以直接和符合人体工程学地解决。他们的行为与他们的完全相同`find`和`findIndex`对应项，除了它们从末尾开始搜索`Array`或 TypedArray。

```js
const inputArray = [{v:1}, {v:2}, {v:3}, {v:4}, {v:5}];
inputArray.findLast((element) => element.v % 2 === 0);
// → {v:4}
inputArray.findLast((element) => element.v % 7 === 0);
// → undefined
inputArray.findLastIndex((element) => element.v % 2 === 0);
// → 3
inputArray.findLastIndex((element) => element.v % 7 === 0);
// → -1
```

## `findLast`和`findLastIndex`支持 { #support }

<feature-support chrome="97"
              firefox="no https://bugzilla.mozilla.org/show_bug.cgi?id=1704385"
              safari="partial https://bugs.webkit.org/show_bug.cgi?id=227939"
              nodejs="no"
              babel="yes https://github.com/zloirock/core-js#array-find-from-last"></feature-support>
