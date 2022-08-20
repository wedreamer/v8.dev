***

## 标题： '堆栈跟踪 API'&#xA;描述： '本文档概述了 V8 的 JavaScript 堆栈跟踪 API。

V8 中引发的所有内部错误在创建时都会捕获堆栈跟踪。这个堆栈跟踪可以从JavaScript通过非标准访问`error.stack`财产。V8 还具有各种钩子，用于控制堆栈跟踪的收集和格式化方式，以及允许自定义错误也收集堆栈跟踪。本文档概述了 V8 的 JavaScript 堆栈跟踪 API。

## 基本堆栈跟踪

默认情况下，几乎所有由 V8 引发的错误都有一个`stack`属性，用于保存最上面的 10 个堆栈帧，格式为字符串。下面是完全格式化的堆栈跟踪的示例：

    ReferenceError: FAIL is not defined
       at Constraint.execute (deltablue.js:525:2)
       at Constraint.recalculate (deltablue.js:424:21)
       at Planner.addPropagate (deltablue.js:701:6)
       at Constraint.satisfy (deltablue.js:184:15)
       at Planner.incrementalAdd (deltablue.js:591:21)
       at Constraint.addConstraint (deltablue.js:162:10)
       at Constraint.BinaryConstraint (deltablue.js:346:7)
       at Constraint.EqualityConstraint (deltablue.js:515:38)
       at chainTest (deltablue.js:807:6)
       at deltaBlue (deltablue.js:879:2)

堆栈跟踪在创建错误时收集，并且无论在何处或多少次引发错误，堆栈跟踪都是相同的。我们收集 10 帧，因为它通常足够有用，但不会太多，以至于它对性能有明显的负面影响。您可以通过设置变量来控制收集的堆栈帧数

```js
Error.stackTraceLimit
```

将其设置为`0`禁用堆栈跟踪收集。任何有限整数值都可以用作要收集的最大帧数。将其设置为`Infinity`意味着收集所有帧。此变量仅影响当前上下文;必须为需要不同值的每个上下文显式设置它。（请注意，在 V8 术语中所谓的“上下文”对应于页面或`<iframe>`在谷歌浏览器）。要设置影响所有上下文的其他默认值，请使用以下 V8 命令行标志：

```bash
--stack-trace-limit <value>
```

要在运行 Google Chrome 时将此标志传递给 V8，请使用：

```bash
--js-flags='--stack-trace-limit <value>'
```

## 异步堆栈跟踪

这`--async-stack-traces`标记（默认情况下处于打开状态，因为[V8 v7.3](https://v8.dev/blog/v8-release-73#async-stack-traces)） 启用新的[零成本异步堆栈跟踪](https://bit.ly/v8-zero-cost-async-stack-traces)，这丰富了`stack`的属性`Error`具有异步堆栈帧的实例，即`await`代码中的位置。这些异步帧标记为`async`在`stack`字符串：

    ReferenceError: FAIL is not defined
        at bar (<anonymous>)
        at async foo (<anonymous>)

在撰写本文时，此功能仅限于`await`地点`Promise.all()`和`Promise.any()`，因为在这些情况下，发动机可以在没有任何额外开销的情况下重建必要的信息（这就是为什么它是零成本的）。

## 自定义异常的堆栈跟踪集合

用于内置错误的堆栈跟踪机制是使用常规堆栈跟踪收集 API 实现的，该 API 也可用于用户脚本。功能

```js
Error.captureStackTrace(error, constructorOpt)
```

将堆栈属性添加到给定的`error`在当时生成堆栈跟踪的对象`captureStackTrace`被调用。通过 收集的堆栈跟踪`Error.captureStackTrace`立即收集，格式化并附加到给定的`error`对象。

可选`constructorOpt`参数允许您传入函数值。收集堆栈跟踪时，对此函数的最顶层调用（包括该调用）上方的所有帧都将被排除在堆栈跟踪之外。这对于隐藏对用户无用的实现详细信息非常有用。定义捕获堆栈跟踪的自定义错误的常用方法是：

```js
function MyError() {
  Error.captureStackTrace(this, MyError);
  // Any other initialization goes here.
}
```

将 MyError 作为第二个参数传入意味着对 MyError 的构造函数调用不会显示在堆栈跟踪中。

## 自定义堆栈跟踪

与 Java 不同，Java 中的异常堆栈跟踪是允许检查堆栈状态的结构化值，而 V8 中的堆栈属性仅保存一个包含格式化堆栈跟踪的平面字符串。除了与其他浏览器的兼容性之外，没有其他原因。但是，这不是硬编码的，而只是默认行为，并且可以由用户脚本覆盖。

对于效率堆栈跟踪，在捕获时不会设置其格式，而是在首次访问堆栈属性时按需设置。堆栈跟踪的格式设置方式为

```js
Error.prepareStackTrace(error, structuredStackTrace)
```

并将此调用返回的任何内容用作`stack`财产。如果将其他函数值分配给`Error.prepareStackTrace`该函数用于设置堆栈跟踪的格式。它传递它正在为其准备堆栈跟踪的错误对象，以及堆栈的结构化表示形式。用户堆栈跟踪格式化程序可以自由地按照他们想要的方式设置堆栈跟踪的格式，甚至返回非字符串值。在调用`prepareStackTrace`完成，以便它也是一个有效的返回值。请注意，自定义`prepareStackTrace`函数只在 堆栈属性的`Error`对象被访问。

结构化堆栈跟踪是`CallSite`对象，每个对象表示一个堆栈帧。一个`CallSite`对象定义以下方法

*   `getThis`：返回的值`this`
*   `getTypeName`：返回的类型`this`作为字符串。 这是存储在 构造函数字段中的函数的名称`this`，如果可用，否则对象的`[[Class]]`内部属性。
*   `getFunction`：返回当前函数
*   `getFunctionName`：返回当前函数的名称，通常为其`name`财产。 如果`name`属性不可用 尝试从函数的上下文中推断名称。
*   `getMethodName`：返回 属性的名称`this`或其包含当前函数的原型之一
*   `getFileName`：如果此函数是在脚本中定义的，则返回脚本的名称
*   `getLineNumber`：如果此函数是在脚本中定义的，则返回当前行号
*   `getColumnNumber`：如果此函数是在脚本中定义的，则返回当前列号
*   `getEvalOrigin`：如果此函数是使用对`eval`返回一个字符串，该字符串表示其中的位置`eval`已调用
*   `isToplevel`：这是顶级调用，也就是说，这是全局对象吗？
*   `isEval`：此调用是否在由调用定义的代码中发生`eval`?
*   `isNative`：此调用是否在本机 V8 代码中？
*   `isConstructor`：这是构造函数调用吗？
*   `isAsync`：这是异步调用（即`await`,`Promise.all()`或`Promise.any()`)?
*   `isPromiseAll`：这是对异步调用`Promise.all()`?
*   `isPromiseAny`：这是对异步调用`Promise.any()`?
*   `getPromiseIndex`：返回 在 中跟随的 promise 元素的索引`Promise.all()`或`Promise.any()`对于异步堆栈跟踪，或`null`如果`CallSite`不是异步`Promise.all()`或`Promise.any()`叫。

默认堆栈跟踪是使用 CallSite API 创建的，因此任何可用的信息也可通过此 API 获得。

为了维护对严格模式函数施加的限制，不允许具有严格模式函数的帧和下面的所有帧（其调用方等）访问其接收器和函数对象。对于这些帧，`getFunction()`和`getThis()`返回`undefined`.

## 兼容性

这里描述的 API 特定于 V8，任何其他 JavaScript 实现都不支持它。大多数实现确实提供了一个`error.stack`属性，但堆栈跟踪的格式可能与此处描述的格式不同。此 API 的推荐用法是：

*   仅当您知道代码在 v8 中运行时，才依赖于格式化堆栈跟踪的布局。
*   设置安全`Error.stackTraceLimit`和`Error.prepareStackTrace`无论哪个实现正在运行您的代码，但请注意，它仅在您的代码在 V8 中运行时才有影响。

## 附录：堆栈跟踪格式

V8 使用的默认堆栈跟踪格式可以为每个堆栈帧提供以下信息：

*   调用是否为构造调用。
*   的类型`this`值 （`Type`).
*   名为 （`functionName`).
*   此函数或其包含函数的原型之一的属性名称 （`methodName`).
*   源中的当前位置 （`location`)

其中任何一个都可能不可用，并且根据这些信息中有多少可用，使用不同的堆栈帧格式。 如果上述所有信息都可用，则格式化的堆栈帧如下所示：

    at Type.functionName [as methodName] (location)

或者，在构造调用的情况下：

    at new functionName (location)

或者，在异步调用的情况下：

    at async functionName (location)

如果只有其中之一`functionName`和`methodName`可用，或者如果它们都可用但相同，则格式为：

    at Type.name (location)

如果两者都不可用`<anonymous>`用作名称。

这`Type`value 是存储在 构造函数字段中的函数的名称`this`.在 V8 中，所有构造函数调用都将此属性设置为构造函数，因此，除非在创建对象后主动更改了此字段，否则它保留其创建它的函数的名称。如果它不可用`[[Class]]`使用对象的属性。

一种特殊情况是全局对象，其中`Type`不显示。在这种情况下，堆栈帧的格式设置为：

    at functionName [as methodName] (location)

该位置本身具有几种可能的格式。最常见的是定义当前函数的脚本中的文件名、行号和列号：

    fileName:lineNumber:columnNumber

如果当前函数是使用`eval`格式为：

    eval at position

...哪里`position`是调用到的完整位置`eval`发生。请注意，这意味着如果对`eval`例如：

    eval at Foo.a (eval at Bar.z (myscript.js:10:3))

如果堆栈帧位于 V8 的库中，则位置为：

    native

...如果不可用，则为：

    unknown location
