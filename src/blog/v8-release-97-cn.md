***

标题： 'V8 版本 v9.7'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-11-05
    标签：
*   释放
    描述： 'V8 版本 v9.7 带来了新的 JavaScript 方法，用于在数组中向后搜索。
    推特： ''

***

每四周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.7)，直到几周后与Chrome 97 Stable合作发布之前，它一直处于测试阶段。V8 v9.7 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `findLast`和`findLastIndex`数组方法

这`findLast`和`findLastIndex`方法`Array`s 和`TypedArray`s 查找与数组末尾的谓词匹配的元素。

例如：

```js
[1,2,3,4].findLast((el) => el % 2 === 0)
// → 4 (last even element)
```

这些方法在没有标志的情况下可用，从 v9.7 开始。

有关更多详细信息，请参阅我们的[功能说明](https://v8.dev/features/finding-in-arrays#finding-elements-from-the-end).

## V8 接口

请使用`git log branch-heads/9.6..branch-heads/9.7 include/v8\*.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.7 -t branch-heads/9.7`以试验 V8 v9.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
