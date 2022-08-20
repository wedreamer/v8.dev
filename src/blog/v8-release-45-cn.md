***

标题： 'V8 发布 v4.5'
作者： 'V8团队'
日期： 2015-07-17 13：33：37
标签：

*   释放
    描述： “V8 v4.5 附带了性能改进，并添加了对多个 ES2015 功能的支持。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 4.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/4.5)，它将处于测试阶段，直到与Chrome 45 Stable协同发布。V8 v4.5 充满了各种面向开发人员的好东西，所以我们想在几周后发布之前，预览一些亮点。

## 改进的 ECMAScript 2015 （ES6） 支持

V8 v4.5 增加了对多个支持[ECMAScript 2015 （ES6）](https://www.ecma-international.org/ecma-262/6.0/)特征。

### 箭头函数

在以下方面的帮助下[箭头函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)可以编写更简化的代码。

```js
const data = [0, 1, 3];
// Code without Arrow Functions
const convertedData = data.map(function(value) { return value * 2; });
console.log(convertedData);
// Code with Arrow Functions
const convertedData = data.map(value => value * 2);
console.log(convertedData);
```

“this”的词法绑定是箭头函数的另一个主要优点。因此，在方法中使用回调变得更加容易。

```js
class MyClass {
  constructor() { this.a = 'Hello, '; }
  hello() { setInterval(() => console.log(this.a + 'World!'), 1000); }
}
const myInstance = new MyClass();
myInstance.hello();
```

### 数组/类型数组函数

所有新方法[数组和类型数组](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array#Methods)ES2015 中指定的现在在 V8 v4.5 中受支持。它们使使用数组和TypedArrays更加方便。添加的方法包括`Array.from`和`Array.of`.镜像最多的方法`Array`还添加了每种TypedArray上的方法。

### `Object.assign`

[`Object.assign`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)使开发人员能够快速合并和克隆对象。

```js
const target = { a: 'Hello, ' };
const source = { b: 'world!' };
// Merge the objects.
Object.assign(target, source);
console.log(target.a + target.b);
```

此功能还可用于混合使用功能。

## 更多的JavaScript语言功能是“可优化的”

多年来，V8的传统优化编译器，[曲轴](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html)，在优化许多常见的 JavaScript 模式方面做得很好。但是，它从未能够支持整个JavaScript语言，并在函数中使用某些语言功能，例如`try`/`catch`和`with`— 将阻止它被优化。V8 必须回退到其较慢的基线编译器来执行该函数。

V8 新优化编译器的设计目标之一，[涡轮风扇](/blog/turbofan-jit)，就是能够最终优化所有的JavaScript，包括ECMAScript 2015的功能。在 V8 v4.5 中，我们开始使用 TurboFan 来优化曲轴不支持的一些语言功能：`for`-`of`,`class`,`with`和计算属性名称。

下面是一个使用“for-of”的代码示例，现在可以由TurboFan编译：

```js
const sequence = ['First', 'Second', 'Third'];
for (const value of sequence) {
  // This scope is now optimizable.
  const object = {a: 'Hello, ', b: 'world!', c: value};
  console.log(object.a + object.b + object.c);
}
```

虽然最初使用这些语言功能的函数不会达到与Crankshaft编译的其他代码相同的峰值性能，但TurboFan现在可以大大超过我们当前的基线编译器。更好的是，随着我们为TurboFan开发更多优化，性能将继续快速提高。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 4.5 -t branch-heads/4.5`以试验 V8 v4.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
