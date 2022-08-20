***

标题： '顶级`await`'
作者： '迈尔斯·博林斯 （[@MylesBorins](https://twitter.com/MylesBorins))'
化身：

*   “迈尔斯-博林斯”
    日期： 2019-10-08
    标签：
*   ECMAScript
*   节点.js 14
    描述： '顶级`await`即将来到 JavaScript 模块！您很快就可以使用`await`无需在异步函数中。
    推文：“1181581262399643650”

***

[顶级`await`](https://github.com/tc39/proposal-top-level-await)使开发人员能够使用`await`异步函数外部的关键字。它的行为就像一个大的异步函数，导致其他模块`import`他们等待，然后再开始评估自己的身体。

## 旧行为 { #old }

什么时候`async`/`await`首次引入，尝试使用`await`在`async`函数导致`SyntaxError`.许多开发人员使用立即调用的异步函数表达式作为访问该功能的一种方式。

```js
await Promise.resolve(console.log('🎉'));
// → SyntaxError: await is only valid in async function

(async function() {
  await Promise.resolve(console.log('🎉'));
  // → 🎉
}());
```

## 新行为 { #new }

具有顶级`await`，上面的代码反而按照您期望的方式工作[模块](/features/modules):

```js
await Promise.resolve(console.log('🎉'));
// → 🎉
```

：：：备注
**注意：**顶级`await` *只*在模块的顶层工作。不支持经典脚本或非异步函数。
:::

## 使用案例

这些用例借用自[规范提案存储库](https://github.com/tc39/proposal-top-level-await#use-cases).

### 动态依赖关系路径

```js
const strings = await import(`/i18n/${navigator.language}`);
```

这允许模块使用运行时值来确定依赖关系。这对于开发/生产拆分、国际化、环境拆分等操作非常有用。

### 资源初始化

```js
const connection = await dbConnector();
```

这允许模块表示资源，并在无法使用模块的情况下产生错误。

### 依赖关系回退

以下示例尝试从 CDN A 加载 JavaScript 库，如果失败，则回退到 CDN B：

```js
let jQuery;
try {
  jQuery = await import('https://cdn-a.example.com/jQuery');
} catch {
  jQuery = await import('https://cdn-b.example.com/jQuery');
}
```

## 模块执行顺序

JavaScript与顶级的最大变化之一`await`是图形中模块的执行顺序。JavaScript 引擎在[后序遍历](https://en.wikibooks.org/wiki/A-level_Computing/AQA/Paper\_1/Fundamentals_of_algorithms/Tree_traversal#Post-order)：从模块图最左侧的子树开始，评估模块，导出它们的绑定，并执行它们的同级，然后是它们的父级。此算法以递归方式运行，直到它执行模块图的根。

在顶级之前`await`，此顺序始终是同步和确定性的：在多次运行代码之间，您的图形保证以相同的顺序执行。曾经是顶级的`await`土地，同样的保证存在，但只要你不使用顶级`await`.

下面是使用顶级时发生的情况`await`在模块中：

1.  当前模块的执行将推迟到已等待的承诺得到解决。
2.  父模块的执行将推迟到`await`及其所有同级，导出绑定。
3.  同级模块和父级模块的同级模块能够以相同的同步顺序继续执行 — 假设没有周期或其他`await`图中包含 ed 承诺。
4.  调用的模块`await`在`await`艾德承诺解决。
5.  父模块和后续树继续以同步顺序执行，只要没有其他`await`艾德承诺。

## 这在 DevTools 中不是已经工作了吗？

确实如此！REPL 在[Chrome DevTools](https://developers.google.com/web/updates/2017/08/devtools-release-notes#await),[节点.js](https://github.com/nodejs/node/issues/13209)和 Safari Web Inspector 支持顶级`await`暂时了。但是，此功能是非标准的，仅限于REPL！它与顶级不同`await`提案，这是语言规范的一部分，仅适用于模块。测试依赖于顶级的生产代码`await`以完全匹配规范提案语义的方式，确保在您的实际应用程序中进行测试，而不仅仅是在DevTools或Node中.js REPL！

## 不是顶级的`await`一把脚枪？

也许你已经看到了[臭名昭著的要点](https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221)由[里奇·哈里斯](https://twitter.com/Rich_Harris)它最初概述了对顶级的一些担忧`await`并敦促JavaScript语言不要实现该功能。一些具体问题是：

*   顶级`await`可能会阻止执行。
*   顶级`await`可能会阻止获取资源。
*   对于CommonJS模块，不会有明确的互操作故事。

该提案的第3阶段版本直接解决了这些问题：

*   由于兄弟姐妹能够执行，因此没有明确的阻止。
*   顶级`await`在模块图的执行阶段发生。此时，所有资源都已获取并链接。不存在阻止获取资源的风险。
*   顶级`await`仅限于模块。明确不支持脚本或 CommonJS 模块。

与任何新的语言功能一样，始终存在意外行为的风险。例如，使用顶级`await`，循环模块依赖关系可能会引入死锁。

没有顶级`await`，JavaScript 开发人员经常使用异步立即调用的函数表达式只是为了获取访问权限`await`.不幸的是，这种模式导致图形执行的确定性和应用程序的静态可分析性降低。由于这些原因，缺乏顶层`await`被视为比该功能引入的危害更高的风险。

## 支持顶级`await`{ #support }

<feature-support chrome="89 https://bugs.chromium.org/p/v8/issues/detail?id=9344"
              firefox="no https://bugzilla.mozilla.org/show_bug.cgi?id=1519100"
              safari="15 https://bugs.webkit.org/show_bug.cgi?id=202484"
              nodejs="14"
              babel="no https://github.com/babel/proposals/issues/44"></feature-support>
