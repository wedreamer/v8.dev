***

标题： 'V8 版本 v7.3'
作者： 'Clemens Backes， compiler wrangler'
化身：

*   克莱门斯-巴克斯
    日期： 2019-02-07 11：30：42
    标签：
*   释放
    描述： 'V8 v7.3 具有 WebAssembly 和异步性能改进、异步堆栈跟踪、Object.fromEntries、String#matchAll 等等！
    推文：“1093457099441561611”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 7.3](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/7.3)，直到几周后与Chrome 73 Stable一起发布，一直处于测试阶段。V8 v7.3 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## 异步堆栈跟踪

我们正在打开[这`--async-stack-traces`旗](/blog/fast-async#improved-developer-experience)默认情况下。[零成本异步堆栈跟踪](https://bit.ly/v8-zero-cost-async-stack-traces)使用高度异步的代码可以更轻松地诊断生产中的问题，因为`error.stack`通常发送到日志文件/服务的属性现在可以更深入地了解导致问题的原因。

## 更快`await`

与上述相关`--async-stack-traces`标记，我们还启用了`--harmony-await-optimization`默认情况下标记，这是`--async-stack-traces`.看[更快的异步功能和承诺](/blog/fast-async#await-under-the-hood)了解更多详情。

## 更快的 Wasm 启动

通过对Liftoff内部的优化，我们显着提高了WebAssembly编译速度，而不会降低所生成代码的质量。对于大多数工作负载，编译时间缩短了 15–25%。

![Liftoff compile time on the Epic ZenGarden demo](/\_img/v8-release-73/liftoff-epic.svg)

## JavaScript 语言特性

V8 v7.3 附带了几个新的 JavaScript 语言功能。

### `Object.fromEntries`

这`Object.entries`API 并不是什么新鲜事：

```js
const object = { x: 42, y: 50 };
const entries = Object.entries(object);
// → [['x', 42], ['y', 50]]
```

不幸的是，没有简单的方法可以从`entries`结果返回到等效对象...直到现在！V8 v7.3 支持[`Object.fromEntries()`](/features/object-fromentries)，这是一个新的内置 API，它执行`Object.entries`:

```js
const result = Object.fromEntries(entries);
// → { x: 42, y: 50 }
```

有关更多信息和示例用例，请参阅[我们`Object.fromEntries`功能说明](/features/object-fromentries).

### `String.prototype.matchAll`

全局 （`g`） 或粘性 （`y`） 正则表达式将其应用于字符串并循环访问所有匹配项。新`String.prototype.matchAll`API 使此操作比以往更容易，特别是对于具有捕获组的正则表达式：

```js
const string = 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev';
const regex = /\b(?<owner>[a-z0-9]+)\/(?<repo>[a-z0-9\.]+)\b/g;

for (const match of string.matchAll(regex)) {
  console.log(`${match[0]} at ${match.index} with '${match.input}'`);
  console.log(`→ owner: ${match.groups.owner}`);
  console.log(`→ repo: ${match.groups.repo}`);
}

// Output:
//
// tc39/ecma262 at 23 with 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev'
// → owner: tc39
// → repo: ecma262
// v8/v8.dev at 36 with 'Favorite GitHub repos: tc39/ecma262 v8/v8.dev'
// → owner: v8
// → repo: v8.dev
```

有关更多详细信息，请阅读[我们`String.prototype.matchAll`解释器](/features/string-matchall).

### `Atomics.notify`

`Atomics.wake`已重命名为`Atomics.notify`匹配[最近的规格更改](https://github.com/tc39/ecma262/pull/1220).

## V8 接口

请使用`git log branch-heads/7.2..branch-heads/7.3 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 7.3 -t branch-heads/7.3`以试验 V8 v7.3 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
