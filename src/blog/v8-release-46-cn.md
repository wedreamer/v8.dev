***

标题： 'V8 发布 v4.6'
作者： 'V8团队'
日期： 2015-08-28 13：33：37
标签：

*   释放
    描述： “V8 v4.6 减少了卡顿，并支持新的 ES2015 语言功能。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 4.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/4.6)，它将处于测试阶段，直到与Chrome 46 Stable协同发布。V8 4.6 充满了各种面向开发人员的好东西，所以我们想在几周后发布之前预览一些亮点。

## 改进的 ECMAScript 2015 （ES6） 支持

V8 v4.6 增加了对多个支持[ECMAScript 2015 （ES6）](https://www.ecma-international.org/ecma-262/6.0/)特征。

### 点差运算符

这[点差运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)使使用数组更加方便。例如，当您只想合并数组时，它使命令式代码过时。

```js
// Merging arrays
// Code without spread operator
const inner = [3, 4];
const merged = [0, 1, 2].concat(inner, [5]);

// Code with spread operator
const inner = [3, 4];
const merged = [0, 1, 2, ...inner, 5];
```

另一个很好的使用点差操作器来代替`apply`:

```js
// Function parameters stored in an array
// Code without spread operator
function myFunction(a, b, c) {
  console.log(a);
  console.log(b);
  console.log(c);
}
const argsInArray = ['Hi ', 'Spread ', 'operator!'];
myFunction.apply(null, argsInArray);

// Code with spread operator
function myFunction (a,b,c) {
  console.log(a);
  console.log(b);
  console.log(c);
}

const argsInArray = ['Hi ', 'Spread ', 'operator!'];
myFunction(...argsInArray);
```

### `new.target`

[`new.target`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new.target)是ES6的功能之一，旨在改善类的使用。在引擎盖下，它实际上是每个函数的隐式参数。如果使用关键字 new 调用函数，则该参数将保留对被调用函数的引用。如果未使用 new，则参数未定义。

在实践中，这意味着您可以使用 new.target 来确定函数是正常调用的，还是通过 new 关键字调用构造函数的。

```js
function myFunction() {
  if (new.target === undefined) {
    throw 'Try out calling it with new.';
  }
  console.log('Works!');
}

// Breaks:
myFunction();

// Works:
const a = new myFunction();
```

使用 ES6 类和继承时，超类构造函数内的 new.target 将绑定到使用 new 调用的派生构造函数。特别是，这使超类能够在构造期间访问派生类的原型。

## 减少卡顿

[叮当](https://en.wiktionary.org/wiki/jank#Noun)可能是一种痛苦，特别是在玩游戏时。通常，当游戏具有多个玩家时，情况会更糟。[oortonline.gl](http://oortonline.gl/)是一个 WebGL 基准测试，它通过渲染具有粒子效果和现代着色器渲染的复杂 3D 场景来测试当前浏览器的局限性。V8团队开始寻求在这些环境中突破Chrome性能的极限。我们还没有完成，但我们努力的成果已经得到了回报。Chrome 46在 oortonline.gl 性能方面取得了令人难以置信的进步，您可以在下面看到自己。

一些优化包括：

*   [类型数组性能改进](https://code.google.com/p/v8/issues/detail?id=3996)
    *   TypedArrays在渲染引擎中被大量使用，例如Turbulenz（oortonline.gl 背后的引擎）。例如，引擎经常在JavaScript中创建类型化数组（如Float32Array），并在应用转换后将它们传递给WebGL。
    *   关键点是优化嵌入器（Blink）和V8之间的交互。
*   [将 TypedArrays 和其他内存从 V8 传递到 Blink 时的性能改进](https://code.google.com/p/chromium/issues/detail?id=515795)
    *   当类型化数组作为单向通信的一部分传递给 WebGL 时，无需为它们创建额外的句柄（V8 也由 V8 跟踪）。
    *   达到外部（Blink）分配的内存限制后，我们现在启动增量垃圾回收，而不是完整的垃圾回收。
*   [空闲垃圾回收计划](/blog/free-garbage-collection)
    *   垃圾回收操作安排在主线程上的空闲时间，这会解除对合成器的阻止，从而实现更流畅的渲染。
*   [为整个旧一代垃圾回收堆启用了并发扫描](https://code.google.com/p/chromium/issues/detail?id=507211)
    *   释放未使用的内存块是在与主线程并发的其他线程上执行的，这大大减少了主垃圾回收暂停时间。

好消息是，与 oortonline.gl 相关的所有更改都是一般的改进，可能会影响大量使用WebGL的应用程序的所有用户。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 4.6 -t branch-heads/4.6`以试验 V8 v4.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
