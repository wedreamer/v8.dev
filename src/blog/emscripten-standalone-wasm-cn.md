***

title： '在 Web 之外：使用 Emscripten 的独立 WebAssembly 二进制文件”
作者： '阿隆扎凯'
化身：

*   'alon-zakai'
    日期： 2019-11-21
    标签：
*   WebAssembly
*   工具
    描述： 'Emscripten 现在支持独立的 Wasm 文件，不需要 JavaScript。
    推文：“1197547645729988608”

***

Emscripten一直首先关注编译到Web和其他JavaScript环境，如Node.js。但随着WebAssembly开始被使用*没有*JavaScript，新的用例正在出现，因此我们一直在努力支持发出[**独立瓦斯姆**](https://github.com/emscripten-core/emscripten/wiki/WebAssembly-Standalone)来自Emscripten的文件，不依赖于Emscripten JS运行时！这篇文章解释了为什么这很有趣。

## 在 Emscripten 中使用独立模式

首先，让我们看看您可以使用此新功能做些什么！似[这篇文章](https://hacks.mozilla.org/2018/01/shrinking-webassembly-and-javascript-code-sizes-in-emscripten/)让我们从一个“hello world”类型的程序开始，该程序导出一个将两个数字相加的单个函数：

```c
// add.c
#include <emscripten.h>

EMSCRIPTEN_KEEPALIVE
int add(int x, int y) {
  return x + y;
}
```

我们通常会用类似的东西来构建它。`emcc -O3 add.c -o add.js`这将发出`add.js`和`add.wasm`.相反，让我们问`emcc`只发出 Wasm：

    emcc -O3 add.c -o add.wasm

什么时候`emcc`看到我们只想要Wasm，然后它使它成为“独立的” - 一个可以尽可能多地自行运行的Wasm文件，没有任何来自Emscripten的JavaScript运行时代码。

反汇编它，它非常小 - 只有87字节！它包含明显的`add`功能

```lisp
(func $add (param $0 i32) (param $1 i32) (result i32)
 (i32.add
  (local.get $0)
  (local.get $1)
 )
)
```

还有一个功能，`_start`,

```lisp
(func $_start
 (nop)
)
```

`_start`是[瓦西](https://github.com/WebAssembly/WASI)spec，Emscripten的独立模式发出它，以便我们可以在WASI运行时中运行。（通常`_start`将进行全局初始化，但在这里我们不需要任何初始化，所以它是空的。

### 编写自己的 JavaScript 加载程序

像这样的独立Wasm文件的一个好处是，您可以编写自定义JavaScript来加载和运行它，根据您的用例，这可能是非常小的。例如，我们可以在 Node 中执行此操作.js：

```js
// load-add.js
const binary = require('fs').readFileSync('add.wasm');

WebAssembly.instantiate(binary).then(({ instance }) => {
  console.log(instance.exports.add(40, 2));
});
```

只需4行！打印的运行`42`不出所料。请注意，虽然这个例子非常简单，但在某些情况下，你根本不需要太多的JavaScript，并且可能比Emscripten的默认JavaScript运行时（它支持一堆环境和选项）做得更好。一个真实的例子是在[zeux 的 meshoptimizer](https://github.com/zeux/meshoptimizer/blob/bdc3006532dd29b03d83dc819e5fa7683815b88e/js/meshopt_decoder.js)- 只有57行，包括内存管理，增长等！

### 在 Wasm 运行时中运行

独立Wasm文件的另一个好处是，您可以在Wasm运行时中运行它们，例如[瓦默](https://wasmer.io),[wasmtime](https://github.com/bytecodealliance/wasmtime)或[瓦维姆](https://github.com/WAVM/WAVM).例如，考虑这个你好世界：

```cpp
// hello.cpp
#include <stdio.h>

int main() {
  printf("hello, world!\n");
  return 0;
}
```

我们可以在任何这些运行时中构建并运行它：

```bash
$ emcc hello.cpp -O3 -o hello.wasm
$ wasmer run hello.wasm
hello, world!
$ wasmtime hello.wasm
hello, world!
$ wavm run hello.wasm
hello, world!
```

Emscripten尽可能多地使用WASI API，因此像这样的程序最终使用100%的WASI，并且可以在支持WASI的运行时中运行（请参阅后面的说明，了解哪些程序需要的不仅仅是WASI）。

### 构建 Wasm 插件

除了 Web 和服务器之外，对于 Wasm 来说，一个令人兴奋的领域是**插件**.例如，图像编辑器可能具有可以对图像执行过滤器和其他操作的 Wasm 插件。对于这种类型的用例，您需要一个独立的Wasm二进制文件，就像到目前为止的示例一样，但它也具有用于嵌入应用程序的适当API。

插件有时与动态库相关，因为动态库是实现它们的一种方法。Emscripten 支持动态库，具有[SIDE_MODULE](https://github.com/emscripten-core/emscripten/wiki/Linking#general-dynamic-linking)选项，这是构建Wasm插件的一种方式。这里描述的新的独立Wasm选项在几个方面都是对此的改进：首先，动态库具有可重定位的内存，如果您不需要它，这将增加开销（如果您在加载后没有将Wasm与另一个Wasm链接，则不会）。其次，如前所述，独立输出也设计为在 Wasm 运行时中运行。

好吧，到目前为止一切顺利：Emscripten可以像往常一样发出JavaScript + WebAssembly，现在它也可以自己发出WebAssembly，这让你可以在没有JavaScript的地方运行它，比如Wasm运行时，或者你可以编写自己的自定义JavaScript加载器代码，等等。现在让我们谈谈背景和技术细节！

## WebAssembly的两个标准API

WebAssembly只能访问它作为导入接收的API - 核心Wasm规范没有具体的API细节。鉴于Wasm的当前轨迹，看起来人们导入和使用API的主要类别将有3种：

*   **网络接口**：这就是Wasm程序在Web上使用的，这是JavaScript也可以使用的现有标准化API。目前这些都是间接调用的，通过JS胶水代码，但将来用[接口类型](https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md)它们将被直接调用。
*   **WASI API**：WASI 专注于标准化服务器上 Wasm 的 API。
*   **其他接口**：各种自定义嵌入将定义自己的特定于应用程序的 API。例如，我们之前给出了一个图像编辑器的例子，其中包含Wasm插件，这些插件实现了一个API来做视觉效果。请注意，插件可能还可以访问“系统”API，就像本机动态库一样，或者它可能非常沙盒化并且根本没有导入（如果嵌入只是调用其方法）。

WebAssembly处于一个有趣的位置，有[两组标准化的 API](https://www.goodreads.com/quotes/589703-the-good-thing-about-standards-is-that-there-are-so).这确实是有道理的，因为一个是用于Web的，一个是针对服务器的，并且这些环境确实有不同的要求;出于类似的原因，Node.js在Web上没有与JavaScript相同的API。

但是，除了Web和服务器之外，还有更多的Wasm插件。首先，插件可以在可能位于Web上的应用程序内运行（就像[JS 插件](https://www.figma.com/blog/an-update-on-plugin-security/#a-technology-change)） 或网络外;另一方面，无论嵌入应用程序在哪里，插件环境都不是Web也不是服务器环境。因此，使用哪些API集并不明显 - 它可能取决于移植的代码，嵌入的Wasm运行时等。

## 让我们尽可能地统一

Emscripten希望在这里提供帮助的一个具体方式是，通过使用WASI API，我们可以尽可能地避免**必要**接口差异。如前所述，在Web上，Emscripten代码通过JavaScript间接访问Web API，因此，如果JavaScript API看起来像WASI，我们将删除不必要的API差异，并且相同的二进制文件也可以在服务器上运行。换句话说，如果Wasm想要记录一些信息，它需要调用JS，如下所示：

```js
wasm   =>   function musl_writev(..) { .. console.log(..) .. }
```

`musl_writev`是 Linux 系统调用接口的实现，[musl libc](https://www.musl-libc.org)用于将数据写入文件描述符，最终调用`console.log`使用适当的数据。Wasm 模块导入并调用`musl_writev`，它定义了 JS 和 Wasm 之间的 ABI。ABI是任意的（事实上，Emscripten随着时间的推移已经改变了它的ABI来优化它）。如果我们将其替换为与WASI匹配的ABI，我们可以得到：

```js
wasm   =>   function __wasi_fd_write(..) { .. console.log(..) .. }
```

这不是一个很大的变化，只是需要对ABI进行一些重构，并且在JS环境中运行时并不重要。但是现在 Wasm 可以在没有 JS 的情况下运行，因为 WASI API 可以被 WASI 运行时识别！这就是之前的独立 Wasm 示例的工作方式，只需重构 Emscripten 即可使用 WASI API。

Emscripten使用WASI API的另一个优点是，我们可以通过查找现实世界的问题来帮助WASI规范。例如，我们发现[更改 WASI 的“位置”常量](https://github.com/WebAssembly/WASI/pull/106)会很有用，我们已经开始了一些讨论[代码大小](https://github.com/WebAssembly/WASI/issues/109)和[POSIX 兼容性](https://github.com/WebAssembly/WASI/issues/122).

尽可能多地使用WASI的Emscripten也很有用，因为它允许用户使用单个SDK来定位Web，服务器和插件环境。Emscripten并不是唯一允许这样做的SDK，因为WASI SDK的输出可以使用[WASI Web Polyfill](https://wasi.dev/polyfill/)或瓦斯默的[wasmer-js](https://github.com/wasmerio/wasmer-js)，但是Emscripten的Web输出更紧凑，因此它允许在不影响Web性能的情况下使用单个SDK。

说到这一点，您可以在单个命令中从Emscripten发出一个独立的Wasscripten文件，其中包含可选的JS：

    emcc -O3 add.c -o add.js -s STANDALONE_WASM

发出`add.js`和`add.wasm`.Wasm文件是独立的，就像以前我们只发出一个Wasm文件本身（`STANDALONE_WASM`当我们说时自动设置`-o add.wasm`），但现在另外还有一个JS文件可以加载并运行它。JS对于在Web上运行它很有用，如果你不想为此编写自己的JS。

## 我们需要吗*非*-独立 Wasm？

为什么`STANDALONE_WASM`标志存在吗？从理论上讲，Emscripten总是可以设置`STANDALONE_WASM`，这样会更简单。但是独立的Wasm文件不能依赖于JS，这有一些缺点：

*   我们无法缩小Wasm导入和导出名称，因为只有在双方同意的情况下，Wasm和加载它的东西才能缩小。
*   通常，我们在JS中创建Wasm内存，以便JS可以在启动期间开始使用它，这使我们能够并行工作。但是在独立的Wasm中，我们必须在Wasm中创建内存。
*   有些 API 在 JS 中很容易做到。例如[`__assert_fail`](https://github.com/emscripten-core/emscripten/pull/9558)，在 C 断言失败时调用，通常为[在 JS 中实现](https://github.com/emscripten-core/emscripten/blob/2b42a35f61f9a16600c78023391d8033740a019f/src/library.js#L1235).它只需要一行，即使你包括它调用的JS函数，总代码大小也很小。另一方面，在独立构建中，我们不能依赖JS，所以我们使用[穆斯尔`assert.c`](https://github.com/emscripten-core/emscripten/blob/b8896d18f2163dbf2fa173694eeac71f6c90b68c/system/lib/libc/musl/src/exit/assert.c#L4).那使用`fprintf`，这意味着它最终会拉入一堆C`stdio`支持，包括具有间接调用的内容，这些调用使难以删除未使用的函数。总体而言，有许多这样的细节最终会对总代码大小产生影响。

如果要在 Web 和其他位置同时运行，并且希望 100% 最佳代码大小和启动时间，则应进行两个单独的生成，其中一个`-s STANDALONE`和一个没有。这很容易，因为它只是翻转一面旗帜！

## 必要的 API 差异

我们看到Emscripten尽可能多地使用WASI API来避免**必要**接口差异。有没有**必要**的？可悲的是，是的 - 一些WASI API需要权衡。例如：

*   WASI不支持各种POSIX功能，例如[用户/组/世界文件权限](https://github.com/WebAssembly/WASI/issues/122)，因此您无法完全实现（Linux）系统`ls`例如（请参阅该链接中的详细信息）。Emscripten现有的文件系统层确实支持其中的一些功能，因此，如果我们切换到WASI API进行所有文件系统操作，那么我们将[失去一些 POSIX 支持](https://github.com/emscripten-core/emscripten/issues/9479#issuecomment-542815711).
*   WASI的`path_open` [具有代码大小的成本](https://github.com/WebAssembly/WASI/issues/109)因为它强制在 Wasm 本身中处理额外的权限。该代码在 Web 上是不必要的。
*   WASI 不提供[用于内存增长的通知 API](https://github.com/WebAssembly/WASI/issues/82)因此，JS 运行时必须在每次导入和导出时不断检查内存是否增长，如果是，则更新其视图。为了避免这种开销，Emscripten提供了一个通知API，`emscripten_notify_memory_growth`哪[你可以看到在一行中实现](https://github.com/zeux/meshoptimizer/blob/bdc3006532dd29b03d83dc819e5fa7683815b88e/js/meshopt_decoder.js#L10)在我们之前提到的zeux的meshoptimizer中。

随着时间的推移，WASI可能会添加更多的POSIX支持，内存增长通知等 - WASI仍然是高度实验性的，预计会有重大变化。目前，为了避免Emscripten中的回归，如果您使用某些功能，我们不会发出100%的WASI二进制文件。特别是，打开文件使用 POSIX 方法而不是 WASI，这意味着如果您调用`fopen`那么生成的Wasm文件将不是100%WASI - 但是，如果您所做的只是使用`printf`，它在已打开的`stdout`，那么它将是100%的WASI，就像我们在开始时看到的“hello world”示例一样，Emscripten的输出确实在WASI运行时中运行。

如果它对用户有用，我们可以添加一个`PURE_WASI`选项会牺牲代码大小以换取严格的WASI合规性，但如果这不是紧急的（到目前为止我们看到的大多数插件用例都不需要完整的文件I / O），那么也许我们可以等待WASI改进到Emscripten可以删除这些非WASI API的位置。这将是最好的结果，正如您在上面的链接中看到的那样，我们正在努力实现这一目标。

但是，即使WASI确实有所改善，也无法避免这样一个事实，即如前所述，Wasm有两个标准化的API。将来，我希望Emscripten将使用接口类型直接调用Web API，因为这将比调用具有WASI外观的JS API更紧凑，然后调用Web API（如`musl_writev`之前的示例）。我们可以有一个polyfill或某种类型的翻译层来帮助这里，但我们不想不必要地使用它，所以我们需要为Web和WASI环境单独构建。（这有点不幸;从理论上讲，如果WASI是Web API的超集，这是可以避免的，但显然这意味着服务器端的妥协。

## 现状

已经有很多工作了！主要限制是：

*   **网络组装限制**：由于Wasm的限制，各种功能，如C++异常，setjmp和pthreads，依赖于JavaScript，并且还没有好的非JS替代品。（Emscripten可能会开始支持其中一些[使用 Asyncify](https://www.youtube.com/watch?v=qQOP6jqZqf8\&list=PLqh1Mztq\_-N2OnEXkdtF5yymcihwqG57y\&index=2\&t=0s)，或者也许我们只是等待[原生 Wasm 功能](https://github.com/WebAssembly/exception-handling/blob/master/proposals/Exceptions.md)以到达 VM。
*   **WASI 限制**：像OpenGL和SDL这样的库和API还没有相应的WASI API。

你**能**仍然在Emscripten的独立模式下使用所有这些，但输出将包含对JS运行时支持代码的调用。因此，它不会是100%的WASI（出于类似的原因，这些功能在WASI SDK中也不起作用）。这些 Wasm 文件不会在 WASI 运行时中运行，但您可以在 Web 上使用它们，也可以为它们编写自己的 JS 运行时。您也可以将它们用作插件;例如，游戏引擎可以具有使用OpenGL渲染的插件，开发人员将在独立模式下编译它们，然后在引擎的Wasm运行时中实现OpenGL导入。独立 Wasm 模式在这里仍然有帮助，因为它使输出像 Emscripten 一样独立。

您可能还会发现**做**有一个我们尚未转换的非JS替代品，因为工作仍在进行中。请[文件错误](https://github.com/emscripten-core/emscripten/issues)，并一如既往地欢迎帮助！
