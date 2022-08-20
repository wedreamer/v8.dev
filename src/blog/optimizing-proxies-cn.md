***

标题： “在 V8 中优化 ES2015 代理”
作者： '玛雅·阿米拉诺娃 （[@Zmayski](https://twitter.com/Zmayski)），代理的优化器
化身：

*   '玛雅-军队'
    日期： 2017-10-05 13：33：37
    标签：
*   ECMAScript
*   基准
*   内部
    描述： '本文解释了 V8 如何提高 JavaScript 代理的性能。
    推文：“915846050447003648”

***

自ES2015以来，代理一直是JavaScript不可或缺的一部分。它们允许拦截对象的基本操作并自定义其行为。代理是项目的核心部分，例如[jsdom](https://github.com/tmpvar/jsdom)和[康林克 RPC 库](https://github.com/GoogleChrome/comlink).最近，我们在V8中投入了大量精力来提高代理的性能。本文对 V8 中的一般性能改进模式，特别是代理的性能改进模式进行了一些阐述。

代理是“用于定义基本操作的自定义行为的对象（例如，属性查找，赋值，枚举，函数调用等）”（定义依据[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)).更多信息可以在[完整规格](https://tc39.es/ecma262/#sec-proxy-objects).例如，以下代码段将日志记录添加到对象上的每个属性访问中：

```js
const target = {};
const callTracer = new Proxy(target, {
  get: (target, name, receiver) => {
    console.log(`get was called for: ${name}`);
    return target[name];
  }
});

callTracer.property = 'value';
console.log(callTracer.property);
// get was called for: property
// value
```

## 构造代理

我们将重点介绍的第一个功能是**建设**代理。我们最初的C++实现一步遵循 ECMAScript 规范，导致 C++ 运行时和 JS 运行时之间至少有 4 次跳转，如下图所示。我们希望将此实现移植到与平台无关的中[CodeStubAssembler](/docs/csa-builtins)（CSA），它在 JS 运行时中执行，而不是在运行时C++执行。此移植可最大程度地减少语言运行时之间的跳转次数。`CEntryStub`和`JSEntryStub`表示下图中的运行时。虚线表示 JS 和运行时C++之间的边界。幸运的是，很多[辅助谓词](https://github.com/v8/v8/blob/4e5db9a6c859df7af95a92e7cf4e530faa49a765/src/code-stub-assembler.h)已经在汇编程序中实现，这使得[初始版本](https://github.com/v8/v8/commit/f2af839b1938b55b4d32a2a1eb6704c49c8d877d#diff-ed49371933a938a7c9896878fd4e4919R97)简洁易读。

下图显示了使用任何代理陷阱调用代理的执行流（在此示例中）`apply`，当代理用作函数时调用，由以下示例代码生成：

```js
function foo(…) { … }
const g = new Proxy({ … }, {
  apply: foo,
});
g(1, 2);
```

![](/\_img/optimizing-proxies/0.png)

将陷阱执行移植到 CSA 后，所有执行都发生在 JS 运行时中，从而将语言之间的跳转次数从 4 次减少到 0 次。

此更改带来了以下性能改进：

![](/\_img/optimizing-proxies/1.png)

我们的 JS 性能得分显示**49% 和 74%**.该分数大致测量给定的微基准标记在1000ms内可以执行多少次。对于某些测试，代码会运行多次，以便在计时器分辨率下获得足够准确的测量值。可以找到以下所有基准测试的代码[在我们的 js-perf-test 目录中](https://github.com/v8/v8/blob/5a5783e3bff9e5c1c773833fa502f14d9ddec7da/test/js-perf-test/Proxies/proxies.js).

## 调用和构造陷阱

下一节将介绍优化调用和构造陷阱（又名[`"apply"`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/apply)“和[`"construct"`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/construct)).

![](/\_img/optimizing-proxies/2.png)

性能改进时*叫*代理很重要 — 最高可达**500%**更快！尽管如此，代理构造的改进还是相当温和的，特别是在没有定义实际陷阱的情况下 - 只有大约**25%**获得。我们通过运行以下命令来调查此问题，并使用[`d8`壳](/docs/build):

```bash
$ out/x64.release/d8 --runtime-call-stats test.js
> run: 120.104000

                      Runtime Function/C++ Builtin        Time             Count
========================================================================================
                                         NewObject     59.16ms  48.47%    100000  24.94%
                                      JS_Execution     23.83ms  19.53%         1   0.00%
                              RecompileSynchronous     11.68ms   9.57%        20   0.00%
                        AccessorNameGetterCallback     10.86ms   8.90%    100000  24.94%
      AccessorNameGetterCallback_FunctionPrototype      5.79ms   4.74%    100000  24.94%
                                  Map_SetPrototype      4.46ms   3.65%    100203  25.00%
… SNIPPET …
```

哪里`test.js`的来源是：

```js
function MyClass() {}
MyClass.prototype = {};
const P = new Proxy(MyClass, {});
function run() {
  return new P();
}
const N = 1e5;
console.time('run');
for (let i = 0; i < N; ++i) {
  run();
}
console.timeEnd('run');
```

事实证明，大部分时间都花在了`NewObject`以及它调用的函数，因此我们开始计划如何在将来的版本中加快速度。

## 获取陷阱

下一节将介绍如何优化其他最常见的操作 — 通过代理获取和设置属性。原来[`get`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/get)trap 比前面的情况更复杂，这是由于 V8 内联缓存的特定行为。有关内联缓存的详细说明，您可以观看[本次演讲](https://www.youtube.com/watch?v=u7zRSm8jzvA).

最终，我们设法将一个工作端口连接到 CSA，结果如下：

![](/\_img/optimizing-proxies/3.png)

登陆更改后，我们注意到Android的大小`.apk`因为Chrome已经成长为**~160KB**，这超出了大约 20 行的帮助程序函数的预期，但幸运的是，我们跟踪了此类统计信息。事实证明，这个函数是从另一个函数调用两次的，该函数被调用3次，从另一个函数调用4次。问题的原因原来是激进的内联。最终，我们将内联函数转换为单独的代码存根，从而节省了宝贵的 KB，从而解决了这个问题 — 最终版本只有**~19KB**增加于`.apk`大小。

## 有陷阱

下一节显示优化的结果[`has`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/has)陷阱。虽然起初我们认为它会更容易（并重用大部分代码的`get`陷阱），原来它有自己的特点。一个特别难以追踪的问题是，当调用`in`算子。实现的改进结果在**71% 和 428%**.同样，在存在陷阱的情况下，增益更为突出。

![](/\_img/optimizing-proxies/4.png)

## 设置陷阱

下一节将讨论如何移植[`set`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/set)陷阱。这一次，我们必须区分[叫](/blog/fast-properties)和索引属性 （[元素](/blog/elements-kinds)).这两种主要类型不是JS语言的一部分，但对于V8的高效属性存储至关重要。最初的实现仍然为元素拯救到运行时，这导致再次跨越语言边界。尽管如此，我们还是实现了以下方面的改进：**27% 和 438%**对于设置疏水阀的情况，代价是减少高达**23%**当它不是。这种性能下降是由于为区分索引属性和命名属性而进行额外检查的开销造成的。对于索引属性，尚无改进。以下是完整的结果：

![](/\_img/optimizing-proxies/5.png)

## 实际使用情况

### 结果来自[jsdom-proxy-benchmark](https://github.com/domenic/jsdom-proxy-benchmark)

jsdom-proxy-benchmark 项目编译[ECMAScript 规范](https://github.com/tc39/ecma262)使用[埃克马可](https://github.com/bterlson/ecmarkup)工具。截至[版本11.2.0](https://github.com/tmpvar/jsdom/blob/master/Changelog.md#1120)，jsdom项目（Ecmarkup的基础）使用代理来实现通用数据结构`NodeList`和`HTMLCollection`.我们使用此基准测试来概述一些比合成微基准测试更实际的用法，并取得了以下结果，平均运行了100次：

*   节点 v8.4.0（无代理优化）：**14277 ± 159 ms**
*   [节点 v9.0.0-v8-金丝雀-20170924](https://nodejs.org/download/v8-canary/v9.0.0-v8-canary20170924898da64843/node-v9.0.0-v8-canary20170924898da64843-linux-x64.tar.gz)（只有一半的陷阱被移植）：**11789 ± 308 ms**
*   速度增加约2.4秒，即**提高约 17%**

![](/\_img/optimizing-proxies/6.png)

*   [转换`NamedNodeMap`使用`Proxy`](https://github.com/domenic/jsdom-proxy-benchmark/issues/1#issuecomment-329047990)处理时间增加
    *   **1.9 秒**在 V8 6.0 上（节点 v8.4.0）
    *   **0.5 秒**在 V8 6.3 （节点 v9.0.0-v8-Canary-20170910）

![](/\_img/optimizing-proxies/7.png)

：：：备注
**注意：**这些结果由[顾婷婷](https://github.com/TimothyGu).谢谢！
:::

### 结果来自[柴.js](https://chaijs.com/)

Chai.js是一个流行的断言库，它大量使用代理。我们通过使用不同版本的 V8 运行测试，创建了一种真实世界的基准测试，大致改进了**超过 4 秒中的 1 秒**，平均 100 次运行：

*   节点 v8.4.0（无代理优化）：**4.2863 ± 0.14 秒**
*   [节点 v9.0.0-v8-金丝雀-20170924](https://nodejs.org/download/v8-canary/v9.0.0-v8-canary20170924898da64843/node-v9.0.0-v8-canary20170924898da64843-linux-x64.tar.gz)（只有一半的陷阱被移植）：**3.1809 ± 0.17 秒**

![](/\_img/optimizing-proxies/8.png)

## 优化方法

我们经常使用通用优化方案来解决性能问题。我们为此特定工作遵循的主要方法包括以下步骤：

*   为特定子功能实施性能测试
*   添加更多规范一致性测试（或从头开始编写）
*   调查原始C++实施
*   将子功能移植到与平台无关的CodeStubAssembler
*   通过手工制作进一步优化代码[涡轮风扇](/docs/turbofan)实现
*   衡量性能改进。

此方法可应用于您可能具有的任何常规优化任务。
