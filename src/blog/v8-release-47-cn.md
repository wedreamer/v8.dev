***

标题： 'V8 发布 v4.7'
作者： 'V8团队'
日期： 2015-10-14 13：33：37
标签：

*   释放
    描述： “V8 v4.7 减少了内存消耗，并支持新的 ES2015 语言功能。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 4.7](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/4.7)，它将处于测试阶段，直到与Chrome 47 Stable协同发布。V8 v4.7 充满了各种面向开发人员的好东西，因此我们想在几周内发布之前预览一些亮点。

## 改进的 ECMAScript 2015 （ES6） 支持

### 休息操作员

这[休息操作员](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/rest_parameters)使开发人员能够将无限数量的参数传递给函数。它类似于`arguments`对象。

```js
// Without rest operator
function concat() {
  var args = Array.prototype.slice.call(arguments, 1);
  return args.join('');
}

// With rest operator
function concatWithRest(...strings) {
  return strings.join('');
}
```

## 支持即将推出的 ES 功能

### `Array.prototype.includes`

[`Array.prototype.includes`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/includes)是一项新功能，目前是 ES2016 中要包含的第 3 阶段提案。它提供了一个简洁的语法，用于通过返回布尔值来确定元素是否在给定数组中。

```js
[1, 2, 3].includes(3); // true
['apple', 'banana', 'cherry'].includes('apple'); // true
['apple', 'banana', 'cherry'].includes('peach'); // false
```

## 在解析时减轻内存压力

[V8 解析器的最新更改](https://code.google.com/p/v8/issues/detail?id=4392)大大减少了使用大型嵌套函数解析文件所消耗的内存。特别是，这允许 V8 运行比以前更大的 asm.js 模块。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 4.7 -t branch-heads/4.7`以试验 V8 v4.7 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
