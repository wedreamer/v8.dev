***

标题： '`globalThis`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-07-16
    标签：
*   ECMAScript
*   ES2020
*   节点.js 12
*   io19
    描述： 'globalThis引入了一种统一的机制来访问任何JavaScript环境中的全局，无论脚本目标如何。
    推文：“1151140681374547969”

***

如果你以前写过 JavaScript 用于 Web 浏览器，那么你可能已经使用过`window`访问全球`this`.在 Node.js 中，您可能使用了`global`.如果您编写的代码必须在任一环境中工作，您可能已经检测到哪些代码可用，然后使用了该代码 - 但是要检查的标识符列表会随着要支持的环境和用例的数量而增长。它很快就会失控：

```js
// A naive attempt at getting the global `this`. Don’t use this!
const getGlobalThis = () => {
  if (typeof globalThis !== 'undefined') return globalThis;
  if (typeof self !== 'undefined') return self;
  if (typeof window !== 'undefined') return window;
  if (typeof global !== 'undefined') return global;
  // Note: this might still return the wrong result!
  if (typeof this !== 'undefined') return this;
  throw new Error('Unable to locate global `this`');
};
const theGlobalThis = getGlobalThis();
```

有关上述方法为何不够充分（以及更复杂的技术）的更多详细信息，请阅读[*一个可怕的`globalThis`通用 JavaScript 中的 polyfill*](https://mathiasbynens.be/notes/globalthis).

[这`globalThis`建议](https://github.com/tc39/proposal-global)介绍一个*统一*访问全局的机制`this`在任何JavaScript环境（浏览器，Node.js或其他东西？）中，无论脚本目标（经典脚本还是模块？）。

```js
const theGlobalThis = globalThis;
```

请注意，新式代码可能不需要访问全局`this`完全。使用JavaScript模块，您可以声明性地`import`和`export`功能，而不是弄乱全局状态。`globalThis`对于 polyfill 和其他需要全局访问的库仍然有用。

## `globalThis`支持 { #support }

<feature-support chrome="71 /blog/v8-release-71#javascript-language-features"
              firefox="65"
              safari="12.1"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-globalthis"></feature-support>
