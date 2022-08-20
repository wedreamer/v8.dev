***

标题： 'V8 发布 v6.0'
作者： 'V8团队'
日期： 2017-06-09 13：33：37
标签：

*   释放
    描述： 'V8 v6.0 附带了多项性能改进，并引入了对`SharedArrayBuffer`s 和对象静止/传播属性。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.0](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.0)，它将处于测试阶段，直到几周后与Chrome 60 Stable协同发布。V8 6.0 充满了各种面向开发人员的好东西。我们想在发布之前预览一些亮点。

## `SharedArrayBuffer`s

V8 v6.0 引入了对[`SharedArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)，这是一种低级机制，用于在 JavaScript worker 之间共享内存，并在 Work 之间同步控制流。SharedArrayBuffers允许JavaScript访问共享内存，原子和futexes。SharedArrayBuffers还解锁了通过asm.js或WebAssembly将线程应用程序移植到Web的能力。

有关简短的低级教程，请参阅规范[教程页面](https://github.com/tc39/ecmascript_sharedmem/blob/master/TUTORIAL.md)或咨询[Emscripten documentation](https://kripken.github.io/emscripten-site/docs/porting/pthreads.html)用于移植 pthreads。

## 对象静止/跨页属性

此版本引入了对象解构分配的 rest 属性和对象文本的展开属性。对象静止/传播属性是 Stage 3 ES.next 功能。

Spread属性还提供了一种简洁的替代方案`Object.assign()`在许多情况下。

```js
// Rest properties for object destructuring assignment:
const person = {
  firstName: 'Sebastian',
  lastName: 'Markbåge',
  country: 'USA',
  state: 'CA',
};
const { firstName, lastName, ...rest } = person;
console.log(firstName); // Sebastian
console.log(lastName); // Markbåge
console.log(rest); // { country: 'USA', state: 'CA' }

// Spread properties for object literals:
const personCopy = { firstName, lastName, ...rest };
console.log(personCopy);
// { firstName: 'Sebastian', lastName: 'Markbåge', country: 'USA', state: 'CA' }
```

有关详细信息，请参阅[我们关于对象静止和传播属性的解释器](/features/object-rest-spread).

## ES2015 性能

V8 v6.0 继续提升 ES2015 功能的性能。此版本包含对语言功能实现的优化，总体上使 V8 的改进大约 10%[阿瑞斯-6](http://browserbench.org/ARES-6/)得分。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.0 -t branch-heads/6.0`以试验 V8 6.0 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
