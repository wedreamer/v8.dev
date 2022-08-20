***

标题： 'V8 附加内容'
作者： 'Domenic Denicola （[@domenic](https://twitter.com/domenic)），流巫师'
化身：

*   “多米尼克-德尼科拉”
    日期： 2016-02-04 13：33：37
    标签：
*   内部
    描述： 'V8 v4.8 包括 'V8 extras'，这是一个简单的接口，旨在允许嵌入者编写高性能的自托管 API。

***

V8 在 JavaScript 本身中实现了 JavaScript 语言的内置对象和函数的很大一部分。例如，您可以看到我们的[承诺实施](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/js/promise.js)是用 JavaScript 编写的。此类内置称为*自托管*.这些实现包含在我们的[启动快照](/blog/custom-startup-snapshots)这样就可以快速创建新上下文，而无需在运行时设置和初始化自托管内置。

V8的嵌入者，如Chromium，有时也希望用JavaScript编写API。这特别适用于独立的平台功能，例如[流](https://streams.spec.whatwg.org/)，或者用于作为“分层平台”的一部分的功能，这些功能建立在预先存在的较低级别功能之上。尽管在启动时总是可以运行额外的代码来引导嵌入 API（例如，在 Node.js中所做的那样），但理想情况下，嵌入器应该能够为其自托管 API 获得与 V8 相同的速度优势。

V8 附加功能是 V8 的新功能，就像我们的[v4.8 发布](/blog/v8-release-48)，旨在允许嵌入者通过简单的接口编写高性能、自托管的 API。附加功能是嵌入器提供的JavaScript文件，这些文件直接编译到V8快照中。他们还可以访问一些帮助程序实用程序，这些实用程序可以更轻松地在JavaScript中编写安全的API。

## 示例

V8 额外文件只是一个具有特定结构的 JavaScript 文件：

```js
(function(global, binding, v8) {
  'use strict';
  const Object = global.Object;
  const x = v8.createPrivateSymbol('x');
  const y = v8.createPrivateSymbol('y');

  class Vec2 {
    constructor(theX, theY) {
      this[x] = theX;
      this[y] = theY;
    }

    norm() {
      return binding.computeNorm(this[x], this[y]);
    }
  }

  Object.defineProperty(global, 'Vec2', {
    value: Vec2,
    enumerable: false,
    configurable: true,
    writable: true
  });

  binding.Vec2 = Vec2;
});
```

这里有一些事情需要注意：

*   这`global`对象不存在于作用域链中，因此对它的任何访问（例如`Object`） 必须通过以下方式明确完成：`global`论点。
*   这`binding`对象是存储嵌入器的值或从中检索值的位置。C++的 API`v8::Context::GetExtrasBindingObject()`提供对`binding`对象从嵌入器的一侧。在我们的玩具示例中，我们让嵌入器执行范数计算;在一个真实的例子中，你可能会委托给嵌入器，以获得一些更棘手的东西，比如URL解析。我们还添加了`Vec2`的构造函数`binding`对象，以便嵌入器代码可以创建`Vec2`实例，而无需经历潜在的可变`global`对象。
*   这`v8`对象提供了少量的 API，以允许您编写安全代码。在这里，我们创建私有符号，以一种无法从外部操纵的方式存储我们的内部状态。（私有符号是 V8 内部概念，在标准 JavaScript 代码中没有意义。V8 的内置功能通常使用“%-function 调用”来表示这类事情，但 V8 附加功能不能使用 %-functions，因为它们是 V8 的内部实现细节，不适合嵌入者依赖。

您可能对这些对象的来源感到好奇。它们全部初始化为[V8 的引导程序](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/bootstrapper.cc)，它安装了一些基本属性，但大部分将初始化留给 V8 的自托管 JavaScript。例如，V8 中几乎每个.js文件都会在`global`;例如，参见[承诺.js](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/js/promise.js\&sq=package:chromium\&l=439)或[乌里.js](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/js/uri.js\&sq=package:chromium\&l=371).我们将 API 安装到`v8`中的对象[一些地方](https://code.google.com/p/chromium/codesearch#search/\&q=extrasUtils\&sq=package:chromium\&type=cs).（该`binding`对象在被 extra 或 embeder 操作之前是空的，所以 V8 本身唯一相关的代码是引导程序创建它的时候。

最后，为了告诉 V8 我们将额外编译，我们在项目的 gypfile 中添加了一行：

```js
'v8_extra_library_files': ['./Vec2.js']
```

（你可以看到一个真实的例子[在 V8 的 gypfile 中](https://code.google.com/p/chromium/codesearch#chromium/src/v8/build/standalone.gypi\&sq=package:chromium\&type=cs\&l=170).)

## V8 附加功能在实践中

V8 附加功能为嵌入器实现功能提供了一种新的轻量级方式。JavaScript代码可以更轻松地操作JavaScript内置，如数组，映射或承诺;它可以在没有仪式的情况下调用其他JavaScript函数;它可以以惯用的方式处理异常。与C++实现不同，通过 V8 附加功能在 JavaScript 中实现的功能可以从内联中受益，并且调用它们不会产生任何边界交叉成本。与Chromium的Web IDL绑定等传统绑定系统相比，这些优势尤其明显。

V8的附加功能在去年被引入和改进，Chromium目前正在使用它们来[实现流](https://code.google.com/p/chromium/codesearch#chromium/src/third_party/WebKit/Source/core/streams/ReadableStream.js).Chromium还在考虑V8的额外功能来实现[滚动自定义](https://codereview.chromium.org/1333323003)和[高效的几何图形接口](https://groups.google.com/a/chromium.org/d/msg/blink-dev/V_bJNtOg0oM/VKbbYs-aAgAJ).

V8的附加功能仍在进行中，并且该界面有一些粗糙的边缘和缺点，我们希望随着时间的推移解决。有改进空间的主要领域是调试故事：错误不容易跟踪，运行时调试通常使用 print 语句完成。在未来，我们希望将V8附加功能集成到Chromium的开发工具和跟踪框架中，无论是对于Chromium本身还是任何使用相同协议的嵌入器。

使用 V8 附加功能时需要注意的另一个原因是编写安全可靠的代码需要额外的开发人员工作。V8 附加代码直接在快照上运行，就像 V8 自托管内置的代码一样。它访问与用户域JavaScript相同的对象，没有绑定层或单独的上下文来防止此类访问。例如，看似简单的东西`global.Object.prototype.hasOwnProperty.call(obj, 5)`有六种可能的方式，由于用户代码修改了内置组件，它可能会失败（计算它们！像Chromium这样的嵌入器需要对任何用户代码保持健壮，无论其行为如何，因此在这样的环境中，编写附加功能比编写传统的C++实现的功能时更需要小心。

如果您想了解有关 V8 附加功能的更多信息，请查看我们的[设计文档](https://docs.google.com/document/d/1AT5-T0aHGp7Lt29vPWFr2-qG8r3l9CByyvKwEuA8Ec0/edit#heading=h.32abkvzeioyz)这涉及到更多细节。我们期待改进 V8 的附加功能，并添加更多功能，使开发人员和嵌入者能够为 V8 运行时编写富有表现力的高性能附加功能。
