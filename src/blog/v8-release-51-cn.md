***

标题： 'V8 发布 v5.1'
作者： 'V8团队'
日期： 2016-04-23 13：33：37
标签：

*   释放
    描述： 'V8 v5.1 带来了性能改进，减少了卡顿和内存消耗，并增加了对 ECMAScript 语言功能的支持。

***

V8 的第一步[发布流程](/docs/release-process)是 Git 主节点的一个新分支，紧接在 Chrome Beta 分支之前的 Chrome Beta 里程碑（大约每六周一次）。我们最新的发布分支是[V8 v5.1](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.1)，它将一直处于测试阶段，直到我们与Chrome 51 Stable一起发布稳定版本。以下是此版本 V8 中面向开发人员的新功能的亮点。

## 改进的 ECMAScript 支持

V8 v5.1 包含许多更改，以符合 ES2017 规范草案。

### `Symbol.species`

数组方法，如`Array.prototype.map`构造子类的实例作为其输出，并可以选择通过更改来自定义此实例[`Symbol.species`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/species).对其他内置类进行类似的更改。

### `instanceof`定制

构造函数可以实现自己的构造函数[`Symbol.hasInstance`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Other_symbols)方法，它重写默认行为。

### 迭代器关闭

作为[`for`-`of`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of)循环（或其他内置迭代，如[传播](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)operator） 现在检查了 close 方法，如果循环提前终止，则会调用该方法。这可用于迭代完成后的清理工作。

### 正则表达式子类`exec`方法

RegExp 子类可以覆盖`exec`仅更改核心匹配算法的方法，并保证由更高级别的函数调用，例如`String.prototype.replace`.

### 函数名称推断

为函数表达式推断的函数名称现在通常在[`name`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/name)函数的属性，遵循这些规则的ES2015形式化。这可能会更改现有的堆栈跟踪，并提供与以前的 V8 版本不同的名称。它还为具有计算属性名称的属性和方法提供有用的名称：

```js
class Container {
  ...
  [Symbol.iterator]() { ... }
  ...
}
const c = new Container;
console.log(c[Symbol.iterator].name);
// → '[Symbol.iterator]'
```

### `Array.prototype.values`

与其他集合类型类似，[`values`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/values)方法`Array`返回对数组内容的迭代器。

## 性能改进

V8 v5.1 还为以下 JavaScript 功能带来了一些显著的性能改进：

*   执行循环，如`for`-`in`
*   `Object.assign`
*   承诺和正则表达式实例化
*   叫`Object.prototype.hasOwnProperty`
*   `Math.floor`,`Math.round`和`Math.ceil`
*   `Array.prototype.push`
*   `Object.keys`
*   `Array.prototype.join`&`Array.prototype.toString`
*   展平重复字符串，例如`'.'.repeat(1000)`

## WebAssembly （Wasm） { #wasm }

V8 v5.1 对[WebAssembly](/blog/webassembly-experimental).您可以通过标志启用它`--expose_wasm`在`d8`.或者，您可以尝试[瓦斯姆演示](https://webassembly.github.io/demo/)与Chrome 51（Beta频道）。<

## 记忆

V8 实现了更多切片[奥里诺科河](/blog/orinoco):

*   平行年轻一代疏散
*   可扩展的记忆集
*   黑色分配

影响是在需要时减少卡顿和内存消耗。

## V8 接口

请查看我们的[API 更改摘要](https://bit.ly/v8-api-changes).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 5.1 -t branch-heads/5.1`以试验 V8 v5.1 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
