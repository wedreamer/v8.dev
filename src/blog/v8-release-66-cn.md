***

标题： 'V8 版本 v6.6'
作者： 'V8团队'
日期： 2018-03-27 13：33：37
标签：

*   释放
    描述： 'V8 v6.6 包括可选的 catch 绑定、扩展的字符串修剪、多项解析/编译/运行时性能改进等等！
    推文：“978534399938584576”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 6.6](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/6.6)，直到几周后与Chrome 66 Stable一起发布之前，它一直处于测试阶段。V8 v6.6 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript 语言特性

### `Function.prototype.toString`修订版 { #function-tostring }

[`Function.prototype.toString()`](/features/function-tostring)现在返回源代码文本的确切切片，包括空格和注释。下面是比较新旧行为的示例：

```js
// Note the comment between the `function` keyword
// and the function name, as well as the space following
// the function name.
function /* a comment */ foo () {}

// Previously:
foo.toString();
// → 'function foo() {}'
//             ^ no comment
//                ^ no space

// Now:
foo.toString();
// → 'function /* comment */ foo () {}'
```

### JSON ⊂ ECMAScript { #json-ecmascript }

现在允许在字符串文本中使用换行符 （U+2028） 和段落分隔符 （U+2029） 符号，[匹配 JSON](/features/subsume-json).以前，这些符号被视为字符串文本中的行终止符，因此使用它们会导致`SyntaxError`例外。

### 自选`catch`绑定 { #optional-捕获绑定 }

这`catch`的子句`try`语句现在可以[不带参数使用](/features/optional-catch-binding).如果您不需要`exception`对象在处理异常的代码中。

```js
try {
  doSomethingThatMightThrow();
} catch { // → Look mom, no binding!
  handleException();
}
```

### 单侧字符串修剪 { #string修剪 }

除了`String.prototype.trim()`，V8 现在实现[`String.prototype.trimStart()`和`String.prototype.trimEnd()`](/features/string-trimming).此功能以前通过非标准提供`trimLeft()`和`trimRight()`方法，这些方法仍保留为新方法的别名，以实现向后兼容性。

```js
const string = '  hello world  ';
string.trimStart();
// → 'hello world  '
string.trimEnd();
// → '  hello world'
string.trim();
// → 'hello world'
```

### `Array.prototype.values`{ #array值 }

[这`Array.prototype.values()`方法](https://tc39.es/ecma262/#sec-array.prototype.values)为数组提供与 ES2015 相同的迭代接口`Map`和`Set`集合：所有集合现在都可以通过以下方式进行迭代`keys`,`values`或`entries`通过调用同名方法。此更改可能与现有的 JavaScript 代码不兼容。如果您在网站上发现奇怪或损坏的行为，请尝试通过以下方式禁用此功能`chrome://flags/#enable-array-prototype-values`和[提交问题](https://bugs.chromium.org/p/v8/issues/entry?template=Defect+report+from+user).

## 执行后的代码缓存

条款*冷*和*热负荷*对于关心加载性能的人来说，这可能是众所周知的。在 V8 中，还有一个概念*热负荷*.让我们以Chrome嵌入V8为例解释一下不同的级别：

*   **冷负荷：**Chrome 会首次看到被访问的网页，并且根本没有缓存任何数据。
*   **热负荷**：Chrome 会记住该网页已被访问过，并且可以从缓存中检索某些资产（例如图像和脚本源文件）。V8 识别出页面已经附带了相同的脚本文件，因此将编译的代码与脚本文件一起缓存在磁盘缓存中。
*   **热负荷**：Chrome 第三次访问网页时，当从磁盘缓存中提供脚本文件时，它还会向 V8 提供在上一次加载期间缓存的代码。V8 可以使用此缓存代码来避免从头开始解析和编译脚本。

在 V8 v6.6 之前，我们在顶级编译后立即缓存生成的代码。V8 仅编译已知在顶级编译期间立即执行的函数，并将其他函数标记为延迟编译。这意味着缓存的代码只包含顶级代码，而所有其他函数必须在每次页面加载时从头开始懒惰地编译。从版本 6.6 开始，V8 缓存脚本顶级执行后生成的代码。当我们执行脚本时，会延迟编译更多函数，并且可以包含在缓存中。因此，这些函数不需要在将来的页面加载时进行编译，从而将热加载方案中的编译和分析时间缩短了 20–60%。可见的用户更改是主线程不那么拥挤，因此加载体验更流畅，更快速。

请尽快查看有关此主题的详细博客文章。

## 背景汇编

一段时间以来，V8已经能够[解析后台线程上的 JavaScript 代码](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html).随着V8的新的[去年出货的点火字节码解释器](/blog/launching-ignition-and-turbofan)，我们能够扩展此支持，以便能够将 JavaScript 源代码编译为后台线程上的字节码。这使得嵌入器能够在主线程上执行更多工作，从而释放它以执行更多的JavaScript并减少卡顿。我们在Chrome 66中启用了此功能，我们看到典型网站上的主线程编译时间减少了5%到20%。有关更多详细信息，请参阅[最近关于此功能的博客文章](/blog/background-compilation).

## 删除 AST 编号

在[去年点火和涡轮风扇发动机发布](/blog/launching-ignition-and-turbofan).我们之前的管道需要一个名为“AST 编号”的后解析阶段，其中生成的抽象语法树中的节点进行了编号，以便使用它的各种编译器具有共同的参考点。

随着时间的推移，这种后处理过程已经膨胀到包括其他功能：为生成器和异步函数编号挂起点，收集内部函数以进行预先编译，初始化文本或检测不可优化的代码模式。

随着新的管道，Ignition字节码成为共同的参考点，并且不再需要编号本身 - 但是，仍然需要剩余的功能，并且AST编号传递仍然存在。

在 V8 v6.6 中，我们终于设法做到了[移出或弃用此剩余功能](https://bugs.chromium.org/p/v8/issues/detail?id=7178)进入其他通道，允许我们移除这棵树的行走。这导致实际编译时间提高了3-5%。

## 异步性能改进

我们设法为承诺和异步函数挤出了一些不错的性能改进，特别是设法缩小了异步函数和去糖化承诺链之间的差距。

![Promise performance improvements](/\_img/v8-release-66/promise.svg)

此外，异步生成器和异步迭代的性能也得到了显著提高，使其成为即将推出的 Node 10 LTS 的可行选择，该 LTS 计划包括 V8 v6.6。例如，考虑以下斐波那契数列实现：

```js
async function* fibonacciSequence() {
  for (let a = 0, b = 1;;) {
    yield a;
    const c = a + b;
    a = b;
    b = c;
  }
}

async function fibonacci(id, n) {
  for await (const value of fibonacciSequence()) {
    if (n-- === 0) return value;
  }
}
```

我们测量了此模式在 Babel 转译之前和之后的以下改进：

![Async generator performance improvements](/\_img/v8-release-66/async-generator.svg)

最后[字节码改进](https://chromium-review.googlesource.com/c/v8/v8/+/866734)到“可挂起函数”，如生成器、异步函数和模块，提高了这些函数在解释器中运行时的性能，并减小了它们的编译大小。我们计划在即将发布的版本中进一步提高异步函数和异步生成器的性能，敬请期待。

## 阵列性能改进

吞吐量性能`Array#reduce`对于多孔双阵列（×[请参阅我们的博客文章，了解什么是多孔和打包数组](/blog/elements-kinds)).这拓宽了以下情况的快速路径：`Array#reduce`适用于多孔和填充的双阵列。

![Array.prototype.reduce performance improvements](/\_img/v8-release-66/array-reduce.svg)

## 不受信任的代码缓解措施

在 V8 v6.6 中，我们登陆[针对侧信道漏洞的更多缓解措施](/docs/untrusted-code-mitigations)以防止信息泄露给不受信任的 JavaScript 和 WebAssembly 代码。

## GYP消失了

这是第一个正式发布没有GYP文件的V8版本。如果您的产品需要已删除的 GYP 文件，则需要将它们复制到您自己的源存储库中。

## 内存分析

Chrome的DevTools现在可以跟踪和快照C++DOM对象，并显示JavaScript中所有可访问的DOM对象及其引用。此功能是 V8 垃圾回收器新的C++跟踪机制的优点之一。欲了解更多信息，请查看[专门的博客文章](/blog/tracing-js-dom).

## V8 接口

请使用`git log branch-heads/6.5..branch-heads/6.6 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 6.6 -t branch-heads/6.6`以试验 V8 v6.6 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
