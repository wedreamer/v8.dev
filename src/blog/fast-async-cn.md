***

标题： “更快的异步函数和承诺”
作者： '玛雅·阿米拉诺娃 （[@Zmayski](https://twitter.com/Zmayski)），永远等待的期待者，以及Benedikt Meurer（[@bmeurer](https://twitter.com/bmeurer)），专业性能保证者
化身：

*   '玛雅-军队'
*   'benedikt-meurer'
    日期： 2018-11-12 16：45：07
    标签：
*   ECMAScript
*   基准
*   介绍
    描述： “更快，更易于调试的异步函数和承诺即将出现在V8 v7.2 / Chrome 72中。
    推文：“1062000102909169670”

***

JavaScript中的异步处理传统上以不是特别快而闻名。更糟糕的是，调试实时JavaScript应用程序（ 特别是Node.js服务器 - 并非易事，*特别是*当涉及到异步编程时。幸运的是，时代在变化。本文探讨了我们如何在 V8 中优化异步函数和承诺（在某种程度上在其他 JavaScript 引擎中也是如此），并描述我们如何改进异步代码的调试体验。

：：：备注
**注意：**如果您更喜欢观看演示文稿而不是阅读文章，那么请欣赏下面的视频！如果没有，请跳过视频并继续阅读。
:::

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/DFP5DKDQfOc" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

## 异步编程的新方法

### 从回调到承诺再到异步函数

在承诺成为JavaScript语言的一部分之前，基于回调的API通常用于异步代码，特别是在Node.js中。下面是一个示例：

```js
function handler(done) {
  validateParams((error) => {
    if (error) return done(error);
    dbQuery((error, dbResults) => {
      if (error) return done(error);
      serviceCall(dbResults, (error, serviceResults) => {
        console.log(result);
        done(error, serviceResults);
      });
    });
  });
}
```

以这种方式使用深度嵌套回调的特定模式通常称为*“回调地狱”*，因为它使代码的可读性降低且难以维护。

幸运的是，现在承诺是JavaScript语言的一部分，同样的代码可以以更优雅和可维护的方式编写：

```js
function handler() {
  return validateParams()
    .then(dbQuery)
    .then(serviceCall)
    .then(result => {
      console.log(result);
      return result;
    });
}
```

最近，JavaScript 获得了对[异步函数](https://developers.google.com/web/fundamentals/primers/async-functions).现在，上述异步代码的编写方式看起来与同步代码非常相似：

```js
async function handler() {
  await validateParams();
  const dbResults = await dbQuery();
  const results = await serviceCall(dbResults);
  console.log(results);
  return results;
}
```

使用异步函数，代码变得更加简洁，并且控制和数据流更容易遵循，尽管执行仍然是异步的。（请注意，JavaScript 执行仍然发生在单个线程中，这意味着异步函数最终不会自己创建物理线程。

### 从事件侦听器回调到异步迭代

另一个在 Node 中特别常见的异步范式.js是[`ReadableStream`s](https://nodejs.org/api/stream.html#stream_readable_streams).下面是一个示例：

```js
const http = require('http');

http.createServer((req, res) => {
  let body = '';
  req.setEncoding('utf8');
  req.on('data', (chunk) => {
    body += chunk;
  });
  req.on('end', () => {
    res.write(body);
    res.end();
  });
}).listen(1337);
```

此代码可能有点难以遵循：传入的数据在只能在回调中访问的块中处理，并且流结束信令也发生在回调中。当您没有意识到函数会立即终止并且实际处理必须在回调中发生时，很容易在此处引入错误。

幸运的是，一个很酷的新的ES2018功能叫做[异步迭代](http://2ality.com/2016/10/asynchronous-iteration.html)可以简化此代码：

```js
const http = require('http');

http.createServer(async (req, res) => {
  try {
    let body = '';
    req.setEncoding('utf8');
    for await (const chunk of req) {
      body += chunk;
    }
    res.write(body);
    res.end();
  } catch {
    res.statusCode = 500;
    res.end();
  }
}).listen(1337);
```

而不是将处理实际请求处理的逻辑放入两个不同的回调中 —`'data'`和`'end'`回调 — 我们现在可以将所有内容放入单个异步函数中，并使用新的`for await…of`循环以异步方式循环访问块。我们还添加了一个`try-catch`阻止以避免`unhandledRejection`问题\[^1]。

\[^1]： 感谢[马泰奥·科利纳](https://twitter.com/matteocollina)将我们指向[此问题](https://github.com/mcollina/make-promises-safe/blob/master/README.md#the-unhandledrejection-problem).

您现在就可以在生产环境中使用这些新功能了！异步函数是**完全支持从 Node.js 8 （V8 v6.2 / Chrome 62） 开始**，而异步迭代器和生成器是**从 Node.js 10 （V8 v6.8 / Chrome 68） 开始完全支持**!

## 异步性能改进

我们已经设法在 V8 v5.5（Chrome 55 & Node.js 7） 和 V8 v6.8 （Chrome 68 & Node.js 10） 之间显著提高了异步代码的性能。我们达到了一个性能水平，开发人员可以安全地使用这些新的编程范例，而不必担心速度。

![](../_img/fast-async/doxbee-benchmark.svg)

上图显示了[doxbee benchmark](https://github.com/v8/promise-performance-tests/blob/master/lib/doxbee-async.js)，用于衡量大量承诺代码的性能。请注意，图表可视化执行时间，这意味着越低越好。

结果[并行基准测试](https://github.com/v8/promise-performance-tests/blob/master/lib/parallel-async.js)，其中特别强调性能[`Promise.all()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)，更令人兴奋：

![](../_img/fast-async/parallel-benchmark.svg)

我们设法改进`Promise.all`性能系数**8×**.

但是，上述基准是合成的微观基准。V8 团队对我们的优化如何影响更感兴趣[实际用户代码的实际性能](/blog/real-world-performance).

![](../_img/fast-async/http-benchmarks.svg)

上图显示了一些流行的HTTP中间件框架的性能，这些框架大量使用承诺和`async`功能。请注意，此图显示每秒的请求数，因此与前面的图表不同，越高越好。这些框架的性能在 Node.js 7 （V8 v5.5） 和 Node.js 10 （V8 v6.8） 之间显著提高。

这些性能改进是三个关键成就的结果：

*   [涡轮风扇](/docs/turbofan)，新的优化编译器 🎉
*   [奥里诺科河](/blog/orinoco)，新的垃圾回收器 🚛
*   一个节点.js 8 个错误导致`await`跳过微分流 🐛

当我们[推出涡轮风扇](/blog/launching-ignition-and-turbofan)在[节点.js 8](https://medium.com/the-node-js-collection/node-js-8-3-0-is-now-available-shipping-with-the-ignition-turbofan-execution-pipeline-aa5875ad3367)，这给整个性能带来了巨大的提升。

我们还一直在研究一个新的垃圾回收器，称为Orinoco，它将垃圾回收工作从主线程中移出，从而显着改善了请求处理。

最后但并非最不重要的一点是，Node中有一个方便的错误.js 8导致`await`在某些情况下跳过微分流，从而获得更好的性能。该错误最初是一个意外的规范冲突，但后来它给了我们优化的想法。让我们从解释错误行为开始：

```js
const p = Promise.resolve();

(async () => {
  await p; console.log('after:await');
})();

p.then(() => console.log('tick:a'))
 .then(() => console.log('tick:b'));
```

上述程序创建了一个已兑现的承诺`p`和`await`的结果，但也将两个处理程序链接到它上面。您期望以哪种顺序`console.log`要执行的调用？

因为`p`已实现，您可能希望它打印`'after:await'`首先，然后是`'tick'`s.事实上，这就是你在 Node.js 8 中得到的行为：

![The await bug in Node.js 8](../_img/fast-async/await-bug-node-8.svg)

尽管此行为看起来很直观，但根据规范，它是不正确的。Node.js 10 实现了正确的行为，即首先执行链接的处理程序，然后才继续执行异步函数。

![Node.js 10 no longer has the await bug](../_img/fast-async/await-bug-node-10.svg)

这*“正确的行为”*可以说不是很明显，实际上让JavaScript开发人员感到惊讶，所以它值得一些解释。在我们深入探讨承诺和异步函数的神奇世界之前，让我们从一些基础开始。

### 任务与微任务

在高层次上有*任务*和*微任务*在 JavaScript 中。任务处理 I/O 和计时器等事件，并一次执行一个。微任务实现 延迟执行`async`/`await`并承诺，并在每个任务结束时执行。在执行返回到事件循环之前，始终清空微任务队列。

![The difference between microtasks and tasks](../_img/fast-async/microtasks-vs-tasks.svg)

有关更多详细信息，请查看Jake Archibald的解释[浏览器中的任务、微任务、队列和计划](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/).Node.js 中的任务模型非常相似。

### 异步函数

根据MDN，异步函数是使用隐式承诺异步操作以返回其结果的函数。异步函数旨在使异步代码看起来像同步代码，向开发人员隐藏异步处理的一些复杂性。

最简单的异步函数如下所示：

```js
async function computeAnswer() {
  return 42;
}
```

当它被调用时，它将返回一个承诺，您可以像任何其他承诺一样获得其价值。

```js
const p = computeAnswer();
// → Promise

p.then(console.log);
// prints 42 on the next turn
```

你只能得到这个承诺的价值`p`下次运行微任务时。换句话说，上述程序在语义上等效于使用`Promise.resolve`替换为：

```js
function computeAnswer() {
  return Promise.resolve(42);
}
```

异步函数的真正强大之处在于`await`表达式，导致函数执行暂停，直到解析承诺，并在实现后恢复。的价值`await`是应验应许的应许。下面是一个示例，显示了这意味着什么：

```js
async function fetchStatus(url) {
  const response = await fetch(url);
  return response.status;
}
```

执行`fetchStatus`在 上挂起`await`，并在`fetch`承诺兑现。这或多或少等同于将处理程序链接到从`fetch`.

```js
function fetchStatus(url) {
  return fetch(url).then(response => response.status);
}
```

该处理程序包含以下代码：`await`在异步函数中。

通常你会通过一个`Promise`自`await`，但实际上您可以等待任何任意 JavaScript 值。如果表达式的值跟在`await`不是承诺，而是转换为承诺。这意味着您可以`await 42`如果你觉得这样做：

```js
async function foo() {
  const v = await 42;
  return v;
}

const p = foo();
// → Promise

p.then(console.log);
// prints `42` eventually
```

更有趣的是，`await`适用于任何[“那么”](https://promisesaplus.com/)，即任何带有`then`方法，即使它不是一个真正的承诺。因此，您可以实现一些有趣的事情，例如测量实际睡眠时间的异步睡眠：

```js
class Sleep {
  constructor(timeout) {
    this.timeout = timeout;
  }
  then(resolve, reject) {
    const startTime = Date.now();
    setTimeout(() => resolve(Date.now() - startTime),
               this.timeout);
  }
}

(async () => {
  const actualTime = await new Sleep(1000);
  console.log(actualTime);
})();
```

让我们看看 V8 的作用`await`在引擎盖下，遵循[规范](https://tc39.es/ecma262/#await).下面是一个简单的异步函数`foo`:

```js
async function foo(v) {
  const w = await v;
  return w;
}
```

调用时，它将包装参数`v`并暂停异步函数的执行，直到解析该承诺。一旦发生这种情况，函数的执行将恢复，并且`w`分配已实现承诺的值。然后从异步函数返回此值。

### `await`在引擎盖下

首先，V8 将此函数标记为*可恢复*，这意味着可以暂停执行，稍后再恢复（在`await`点）。然后它创造了所谓的`implicit_promise`，这是调用异步函数时返回的 promise，它最终解析为 async 函数生成的值。

![Comparison between a simple async function and what the engine turns it into](../_img/fast-async/await-under-the-hood.svg)

然后是有趣的一点：实际`await`.首先将值传递给`await`被包裹成一个承诺。然后，将处理程序附加到此包装的 promise，以便在 promise 实现后恢复该函数，并暂停 async 函数的执行，返回`implicit_promise`给调用方。一旦`promise`完成，则使用值恢复异步函数的执行`w`从`promise`，以及`implicit_promise`通过`w`.

简而言之，初步步骤`await v`是：

1.  包装`v`— 传递给的值`await`—— 变成一个承诺。
2.  附加处理程序以便稍后恢复异步函数。
3.  暂停异步函数并返回`implicit_promise`给调用方。

让我们逐步完成各个操作。假设正在被的东西`await`ed已经是一个承诺，它的价值已经实现`42`.然后，引擎将创建一个新的`promise`并用任何存在的东西解决这个问题`await`编辑。这在下一轮时将这些承诺的延迟链接，通过规范所称的[`PromiseResolveThenableJob`](https://tc39.es/ecma262/#sec-promiseresolvethenablejob).

![](../_img/fast-async/await-step-1.svg)

然后引擎创建另一个所谓的`throwaway`承诺。它被称为*一次性*因为没有任何东西被束缚在它上面 - 它完全是引擎内部的。这`throwaway`然后，承诺被链接到`promise`，并使用适当的处理程序恢复异步函数。这`performPromiseThen`操作本质上是什么[`Promise.prototype.then()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then)确实如此，在幕后。最后，暂停异步函数的执行，并将控制权返回给调用方。

![](../_img/fast-async/await-step-2.svg)

调用方中的执行继续，最终调用堆栈变为空。然后JavaScript引擎开始运行微任务：它运行先前计划的内容[`PromiseResolveThenableJob`](https://tc39.es/ecma262/#sec-promiseresolvethenablejob)，这将安排新的[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob)以链接`promise`到传递给的值上`await`.然后，引擎返回到处理微任务队列，因为在继续主事件循环之前必须清空微任务队列。

![](../_img/fast-async/await-step-3.svg)

接下来是[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob)，它满足`promise`从承诺的价值，我们是`await`ing —`42`在这种情况下 — 并将反应安排到`throwaway`承诺。然后，引擎再次返回到微任务循环，其中包含要处理的最终微任务。

![](../_img/fast-async/await-step-4-final.svg)

现在这一秒[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob)将分辨率传播到`throwaway`promise，并恢复异步函数的挂起执行，返回值`42`从`await`.

![Summary of the overhead of await](../_img/fast-async/await-overhead.svg)

总结我们学到的东西，对于每个`await`引擎必须创建**两个额外的**承诺（即使右手边已经是承诺）并且它需要**至少三个**微任务队列滴答声。谁知道单身`await`表达式导致*那么多的开销*?!

![](../_img/fast-async/await-code-before.svg)

让我们来看看这个开销来自哪里。第一行负责创建包装器承诺。第二行立即解析包装器承诺与`await`ed 值`v`.这两条线负责一个额外的承诺加上三个微分中的两个。如果`v`已经是一个承诺（这是常见的情况，因为应用程序通常`await`承诺）。在不太可能的情况下，开发人员`await`s 上，例如`42`，引擎仍然需要将其包装成一个承诺。

事实证明，已经有一个[`promiseResolve`](https://tc39.es/ecma262/#sec-promise-resolve)规范中仅在需要时执行包装的操作：

![](../_img/fast-async/await-code-comparison.svg)

此操作返回未更改的 promise，并且仅在必要时将其他值包装到 promise 中。通过这种方式，您可以保存其中一个附加承诺，加上微任务队列上的两个刻度线，用于将值传递给的常见情况`await`已经是一个承诺。此新行为已[在 V8 v7.2 中默认启用](/blog/v8-release-72#async%2Fawait).对于 V8 v7.1，可以使用`--harmony-await-optimization`旗。我们已经[建议对 ECMAScript 规范进行此更改](https://github.com/tc39/ecma262/pull/1250)也。

以下是新的和改进的方法`await`幕后工作，一步一步：

![](../_img/fast-async/await-new-step-1.svg)

让我们再次假设我们`await`一个承诺，已经实现`42`.多亏了神奇的[`promiseResolve`](https://tc39.es/ecma262/#sec-promise-resolve)这`promise`现在只是指同样的应许`v`，因此此步骤中无需执行任何操作。之后，引擎继续像以前一样，创建`throwaway`承诺，调度一个[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob)在微任务队列的下一个 tick 上恢复异步函数，暂停函数的执行，然后返回调用方。

![](../_img/fast-async/await-new-step-2.svg)

然后，最终当所有JavaScript执行完成时，引擎开始运行微任务，因此它执行[`PromiseReactionJob`](https://tc39.es/ecma262/#sec-promisereactionjob).此作业传播分辨率`promise`自`throwaway`，然后恢复异步函数的执行，从而产生`42`从`await`.

![Summary of the reduction in await overhead](../_img/fast-async/await-overhead-removed.svg)

此优化避免了在将值传递给`await`已经是一个承诺，在这种情况下，我们从最低限度**三**微分流到只是**一**微滴答。此行为类似于 Node.js 8 所做的，只是现在它不再是一个错误 - 它现在是一个正在标准化的优化！

它仍然感觉不对，引擎必须创建这个`throwaway`承诺，尽管完全是引擎的内部。事实证明，`throwaway`承诺只是为了满足内部的API约束`performPromiseThen`规范中的操作。

![](../_img/fast-async/await-optimized.svg)

最近在一个[编辑更改](https://github.com/tc39/ecma262/issues/694)到 ECMAScript 规范。引擎不再需要创建`throwaway`承诺`await`— 大多数时候\[^2]。

\[^2]：V8 仍然需要创建`throwaway`承诺如果[`async_hooks`](https://nodejs.org/api/async_hooks.html)正在 Node.js 中使用，因为`before`和`after`钩子在*上下文*的`throwaway`承诺。

![Comparison of await code before and after the optimizations](../_img/fast-async/node-10-vs-node-12.svg)

比较`await`在节点中.js 10 到优化`await`这很可能在 Node 中.js 12 显示了此更改对性能的影响：

![](../_img/fast-async/benchmark-optimization.svg)

**`async`/`await`现在优于手写的承诺代码**.这里的关键是，我们通过修补规范，显著降低了异步函数的开销 ——不仅在 V8 中，而且在所有 JavaScript 引擎中。

**更新：**从 V8 v7.2 和 Chrome 72 开始，`--harmony-await-optimization`默认情况下处于启用状态。[补丁](https://github.com/tc39/ecma262/pull/1250)到 ECMAScript 规范被合并。

## 改进的开发人员体验

除了性能之外，JavaScript开发人员还关心诊断和修复问题的能力，这在处理异步代码时并不总是那么容易。[Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools)支持*异步堆栈跟踪*，即堆栈跟踪，其中不仅包括堆栈的当前同步部分，还包括异步部分：

![](../_img/fast-async/devtools.png)

这是本地开发过程中非常有用的功能。但是，一旦部署了应用程序，此方法并不能真正帮助您。在事后调试期间，您只会看到`Error#stack`输出在日志文件中，这并不能告诉您有关异步部分的任何信息。

我们最近一直在努力[*零成本异步堆栈跟踪*](https://bit.ly/v8-zero-cost-async-stack-traces)这丰富了`Error#stack`具有异步函数调用的属性。“零成本”听起来很令人兴奋，不是吗？当Chrome DevTools功能带来重大开销时，它怎么可能是零成本的？考虑此示例，其中`foo`调用`bar`异步，以及`bar`在以下情况下引发异常`await`承诺：

```js
async function foo() {
  await bar();
  return 42;
}

async function bar() {
  await Promise.resolve();
  throw new Error('BEEP BEEP');
}

foo().catch(error => console.log(error.stack));
```

在 Node.js 8 或 Node.js 10 中运行此代码会产生以下输出：

```text/2
$ node index.js
Error: BEEP BEEP
    at bar (index.js:8:9)
    at process._tickCallback (internal/process/next_tick.js:68:7)
    at Function.Module.runMain (internal/modules/cjs/loader.js:745:11)
    at startup (internal/bootstrap/node.js:266:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:595:3)
```

请注意，尽管调用`foo()`导致错误，`foo`根本不是堆栈跟踪的一部分。这使得 JavaScript 开发人员很难执行事后调试，而不管你的代码是部署在 Web 应用程序中还是部署在某些云容器中。

这里有趣的是，引擎知道它必须在哪里继续`bar`完成：紧接在`await`在函数中`foo`.巧合的是，这也是函数的地方`foo`已暂停。引擎可以使用此信息来重建部分异步堆栈跟踪，即`await`网站。进行此更改后，输出变为：

```text/2,7
$ node --async-stack-traces index.js
Error: BEEP BEEP
    at bar (index.js:8:9)
    at process._tickCallback (internal/process/next_tick.js:68:7)
    at Function.Module.runMain (internal/modules/cjs/loader.js:745:11)
    at startup (internal/bootstrap/node.js:266:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:595:3)
    at async foo (index.js:2:3)
```

在堆栈跟踪中，最上面的函数排在第一位，然后是同步堆栈跟踪的其余部分，然后是异步调用`bar`在函数中`foo`.此更改在 V8 中实现，在新的`--async-stack-traces`旗。**更新**：从 V8 v7.3 开始，`--async-stack-traces`默认情况下处于启用状态。

但是，如果您将其与上面的 Chrome DevTools 中的异步堆栈跟踪进行比较，您会注意到实际的调用站点`foo`在堆栈跟踪的异步部分中丢失。如前所述，这种方法利用了以下事实：`await`恢复和暂停位置是相同的 - 但对于常规[`Promise#then()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then)或[`Promise#catch()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)电话，事实并非如此。有关更多背景信息，请参阅 Mathias Bynens 的解释[为什么`await`打`Promise#then()`](https://mathiasbynens.be/notes/async-stack-traces).

## 结论

由于两项重要的优化，我们使异步函数更快：

*   去除两个额外的微孔，以及
*   删除`throwaway`承诺。

最重要的是，我们通过以下方式改进了开发人员体验：[*零成本异步堆栈跟踪*](https://bit.ly/v8-zero-cost-async-stack-traces)，适用于`await`在异步函数中，并且`Promise.all()`.

我们还为JavaScript开发人员提供了一些不错的性能建议：

*   喜爱`async`函数和`await`通过手写的承诺代码，以及
*   坚持使用JavaScript引擎提供的本机承诺实现，以从快捷方式中受益，即避免两个microticks`await`.
