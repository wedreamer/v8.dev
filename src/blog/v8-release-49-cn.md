***

标题： 'V8 发布 v4.9'
作者： 'V8团队'
日期： 2016-01-26 13：33：37
标签：

*   释放
    描述： 'V8 v4.9 带有改进`Math.random`实现并增加了对几个新的ES2015语言功能的支持。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 4.9](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/4.9)，它将处于测试阶段，直到与Chrome 49 Stable协同发布。V8 4.9 充满了各种面向开发人员的好东西，所以我们想在几周后发布之前预览一些亮点。

## 91% 的 ECMAScript 2015 （ES6） 支持

在 V8 版本 4.9 中，我们交付的 JavaScript ES2015 功能比任何其他先前版本都多，使我们完成了 91%。[康加克斯兼容性表](https://kangax.github.io/compat-table/es6/)（截至1月26日）。V8 现在支持解构、默认参数、代理对象和反射 API。4.9 版还使块级构造如`class`和`let`在严格模式之外可用，并在正则表达式上增加了对粘性标志的支持，并且可自定义`Object.prototype.toString`输出。

### 解构

变量声明、参数和赋值现在支持[解构](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)通过模式的对象和数组。例如：

```js
const o = {a: [1, 2, 3], b: {p: 4}, c: {q: 5}};
let {a: [x, y], b: {p}, c, d} = o;              // x=1, y=2, p=4, c={q: 5}
[x, y] = [y, x];                                // x=2, y=1
function f({a, b}) { return [a, b]; }
f({a: 4});                                      // [4, undefined]
```

数组模式可以包含分配给数组其余部分的 rest 模式：

```js
const [x, y, ...r] = [1, 2, 3, 4];              // x=1, y=2, r=[3,4]
```

此外，可以为模式元素指定默认值，以便在相应属性不匹配的情况下使用这些值：

```js
const {a: x, b: y = x} = {a: 4};                // x=4, y=4
// or…
const [x, y = 0, z = 0] = [1, 2];               // x=1, y=2, z=0
```

解构可用于使从对象和数组访问数据更加紧凑。

### 代理和反射

经过多年的发展，V8现在完全实现了[代理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)，最新符合 ES2015 规范。代理是一种强大的机制，用于通过开发人员提供的一组钩子来自定义属性访问来虚拟化对象和函数。除了对象虚拟化之外，代理还可用于实现拦截，添加属性设置的验证，简化调试和分析，以及解锁高级抽象，例如[膜](http://tvcutsem.github.io/js-membranes/).

若要代理对象，必须创建一个定义各种陷阱的处理程序占位符对象，并将其应用于代理虚拟化的目标对象：

```js
const target = {};
const handler = {
  get(target, name='world') {
    return `Hello, ${name}!`;
  }
};

const foo = new Proxy(target, handler);
foo.bar;
// → 'Hello, bar!'
```

Proxy 对象附带 Reflect 模块，该模块为所有代理陷阱定义了合适的默认值：

```js
const debugMe = new Proxy({}, {
  get(target, name, receiver) {
    console.log(`Debug: get called for field: ${name}`);
    return Reflect.get(target, name, receiver);
  },
  set(target, name, value, receiver) {
    console.log(`Debug: set called for field: ${name}, and value: ${value}`);
    return Reflect.set(target, name, value, receiver);
  }
});

debugMe.name = 'John Doe';
// Debug: set called for field: name, and value: John Doe
const title = `Mr. ${debugMe.name}`; // → 'Mr. John Doe'
// Debug: get called for field: name
```

有关代理和反射 API 用法的详细信息，请参阅[MDN代理页面](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#Examples).

### 默认参数

在 ES5 及更低版本中，函数定义中的可选参数需要样板代码来检查参数是否未定义：

```js
function sublist(list, start, end) {
  if (typeof start === 'undefined') start = 0;
  if (typeof end === 'undefined') end = list.length;
  ...
}
```

ES2015 现在允许函数参数具有[默认值](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters)，提供更清晰、更简洁的函数定义：

```js
function sublist(list, start = 0, end = list.length) { … }
sublist([1, 2, 3], 1);
// sublist([1, 2, 3], 1, 3)
```

当然，默认参数和解构可以组合使用：

```js
function vector([x, y, z] = []) { … }
```

### 草率模式下的类和词法声明

V8 支持词法声明 （`let`,`const`、块-本地`function`） 和分别从版本 4.1 和 4.2 开始的类，但到目前为止，为了使用它们，需要严格的模式。从 V8 版本 4.9 开始，根据 ES2015 规范，所有这些功能现在都在严格模式之外启用。这使得 DevTools 控制台中的原型设计变得更加容易，尽管我们通常鼓励开发人员将新代码升级到严格模式。

### 正则表达式

V8 现在支持新的[粘性旗帜](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky)在正则表达式上。粘性标志切换字符串中的搜索是从字符串的开头开始（正常）还是从`lastIndex`属性（粘性）。此行为对于使用许多不同的正则表达式有效地分析任意长的输入字符串非常有用。要启用粘性搜索，请添加`y`标志为正则表达式：（例如`const regex = /foo/y;`).

### 定制`Object.prototype.toString`输出

用`Symbol.toStringTag`，用户定义的类型现在可以在传递给`Object.prototype.toString`（直接或由于字符串强制）：

```js
class Custom {
  get [Symbol.toStringTag]() {
    return 'Custom';
  }
}
Object.prototype.toString.call(new Custom);
// → '[object Custom]'
String(new Custom);
// → '[object Custom]'
```

## 改进`Math.random()`

V8 v4.9 包括对`Math.random()`.[正如上个月宣布的](/blog/math-random)，我们将 V8 的 PRNG 算法切换为[异速128+](http://vigna.di.unimi.it/ftp/papers/xorshiftplus.pdf)以提供更高质量的伪随机性。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 4.9 -t branch-heads/4.9`以试验 V8 v4.9 中的新功能。或者，您可以订阅[Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
