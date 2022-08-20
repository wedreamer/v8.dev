***

标题： 'WebAssembly integration with JavaScript BigInt'
作者： '阿隆扎凯'
化身：

*   'alon-zakai'
    日期： 2020-11-12
    标签：
*   WebAssembly
*   ECMAScript
    描述： 'BigInts使得在JavaScript和WebAssembly之间传递64位整数变得容易。这篇文章解释了这意味着什么以及为什么它很有用，其中包括让开发人员更简单，让代码运行得更快，以及加快构建时间。
    推文：“1331966281571037186”

***

这[JS-BigInt-Integration](https://github.com/WebAssembly/JS-BigInt-integration)功能使得在JavaScript和WebAssembly之间传递64位整数变得容易。这篇文章解释了这意味着什么以及为什么它很有用，其中包括让开发人员更简单，让代码运行得更快，以及加快构建时间。

## 64 位整数

JavaScript Numbers 是双精度值，即 64 位浮点值。此类值可以包含任何具有完全精度的 32 位整数，但不能包含所有 64 位整数。另一方面，WebAssembly完全支持64位整数，`i64`类型。连接两者时出现问题：例如，如果 Wasm 函数返回 i64，则 VM 会引发异常（如果从 JavaScript 调用它），如下所示：

    TypeError: Wasm function signature contains illegal type

正如错误所说，`i64`不是 JavaScript 的合法类型。

从历史上看，最好的解决方案是将Wasm“合法化”。合法化意味着将 Wasm 导入和导出转换为使用 JavaScript 的有效类型。在实践中，这做了两件事：

1.  将一个 64 位整数参数替换为两个 32 位整数参数，分别表示低位和高位。
2.  将 64 位整数返回值替换为表示低位的 32 位整数返回值，并在侧面使用 32 位值作为高位。

例如，考虑以下 Wasm 模块：

```wasm
(module
  (func $send_i64 (param $x i64)
    ..))
```

合法化将把它变成这样：

```wasm
(module
  (func $send_i64 (param $x_low i32) (param $x_high i32)
    (local $x i64) ;; the real value the rest of the code will use
    ;; code to combine $x_low and $x_high into $x
    ..))
```

合法化是在工具端完成的，然后到达运行它的 VM。例如，[二进制](https://github.com/WebAssembly/binaryen)工具链库有一个称为[合法化JSInterface](https://github.com/WebAssembly/binaryen/blob/fd7e53fe0ae99bd27179cb35d537e4ce5ec1fe11/src/passes/LegalizeJSInterface.cpp)执行该转换，该转换在[Emscripten](https://emscripten.org/)当需要时。

## 合法化的缺点

合法化在许多方面都足够好，但它确实有缺点，比如将32位片段组合或拆分为64位值的额外工作。虽然这种情况很少发生在炎热的路径上，但当它发生时，减速可能会很明显 - 我们稍后会看到一些数字。

另一个烦恼是，合法化是用户注意到的，因为它改变了JavaScript和Wasm之间的接口。下面是一个示例：

```c
// example.c

#include <stdint.h>

extern void send_i64_to_js(int64_t);

int main() {
  send_i64_to_js(0xABCD12345678ULL);
}
```

```javascript
// example.js

mergeInto(LibraryManager.library, {
  send_i64_to_js: function(value) {
    console.log("JS received: 0x" + value.toString(16));
  }
});
```

这是一个很小的C程序，它调用[JavaScript 库](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#implement-c-in-javascript)函数（也就是说，我们在C中定义了一个extern C函数，并在JavaScript中实现它，作为在Wasm和JavaScript之间调用的简单而低级的方式）。这个程序所做的就是发送一个`i64`到JavaScript，我们试图打印它。

我们可以用

    emcc example.c --js-library example.js -o out.js

当我们运行它时，我们没有得到我们期望的：

    node out.js
    JS received: 0x12345678

我们发送`0xABCD12345678`但我们只收到了`0x12345678`😔.这里发生的事情是，合法化使`i64`一分为二`i32`s，我们的代码刚刚收到低32位，并忽略了发送的另一个参数。为了正确处理事情，我们需要做这样的事情：

```javascript
  // The i64 is split into two 32-bit parameters, “low” and “high”.
  send_i64_to_js: function(low, high) {
    console.log("JS received: 0x" + high.toString(16) + low.toString(16));
  }
```

现在运行这个，我们得到

    JS received: 0xabcd12345678

如您所见，有可能接受合法化。但它可能有点烦人！

## 解决方案：JavaScript BigInts

JavaScript 有[大英特](/features/bigint)值现在，它们表示任意大小的整数，因此它们可以正确表示 64 位整数。想要用它们来表示是很自然的`i64`来自瓦斯姆。这正是JS-BigInt-Integration功能所做的！

Emscripten支持Wasm BigInt集成，我们可以使用它来编译原始示例（没有任何合法化的黑客），只需添加`-s WASM_BIGINT`:

    emcc example.c --js-library example.js -o out.js -s WASM_BIGINT

然后我们可以运行它（请注意，我们需要传递Node.js一个标志来启用BigInt集成当前）：

    node --experimental-wasm-bigint a.out.js
    JS received: 0xabcd12345678

完美，正是我们想要的！

这不仅更简单，而且更快。如前所述，在实践中，很少`i64`转换发生在热路径上，但是当它发生时，减速可能会很明显。如果我们把上面的例子变成一个基准测试，运行许多调用`send_i64_to_js`，则 BigInt 版本的速度提高了 18%。

BigInt集成的另一个好处是工具链可以避免合法化。如果Emscripten不需要合法化，那么它可能在LLVM发出的Wasm上没有任何工作要做，这加快了构建时间。如果您使用`-s WASM_BIGINT`并且不提供任何其他需要进行更改的标志。例如`-O0 -s WASM_BIGINT`有效（但优化的构建[运行 Binaryen optimizer](https://emscripten.org/docs/optimizing/Optimizing-Code.html#link-times)这对大小很重要）。

## 结论

WebAssembly BigInt 集成已在[多个浏览器](https://webassembly.org/roadmap/)，包括Chrome 85（2020-08-25发布），因此您可以立即试用！
