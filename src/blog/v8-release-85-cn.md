***

标题： 'V8 发布 v8.5'
作者：“Zeynep Cankara，跟踪一些地图”
化身：

*   'zeynep-cankara'
    日期： 2020-07-21
    标签：
*   释放
    描述： 'V8 release v8.5 具有 Promise.any、String#replaceAll、逻辑赋值运算符、WebAssembly 多值和 BigInt 支持以及性能改进。
    鸣叫：

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 8.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.5)，直到几周后与Chrome 85 Stable合作发布之前，它一直处于测试阶段。V8 v8.5 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### `Promise.any`和`AggregateError`

`Promise.any`是一个承诺组合器，一旦满足其中一个输入承诺，它就会解析生成的承诺。

```js
const promises = [
  fetch('/endpoint-a').then(() => 'a'),
  fetch('/endpoint-b').then(() => 'b'),
  fetch('/endpoint-c').then(() => 'c'),
];
try {
  const first = await Promise.any(promises);
  // Any of the promises was fulfilled.
  console.log(first);
  // → e.g. 'b'
} catch (error) {
  // All of the promises were rejected.
  console.assert(error instanceof AggregateError);
  // Log the rejection values:
  console.log(error.errors);
}
```

如果所有输入承诺都被拒绝，则生成的承诺将被拒绝，并带有`AggregateError`包含`errors`保存拒绝值数组的属性。

请参阅[我们的解释员](https://v8.dev/features/promise-combinators#promise.any)了解更多。

### `String.prototype.replaceAll`

`String.prototype.replaceAll`提供了一种简单的方法来替换所有出现的子字符串，而无需创建全局`RegExp`.

```js
const queryString = 'q=query+string+parameters';

// Works, but requires escaping inside regular expressions.
queryString.replace(/\+/g, ' ');
// → 'q=query string parameters'

// Simpler!
queryString.replaceAll('+', ' ');
// → 'q=query string parameters'
```

请参阅[我们的解释员](https://v8.dev/features/string-replaceall)了解更多。

### 逻辑赋值运算符

逻辑赋值运算符是组合逻辑运算的新的复合赋值运算符`&&`,`||`或`??`与分配。

```js
x &&= y;
// Roughly equivalent to x && (x = y)
x ||= y;
// Roughly equivalent to x || (x = y)
x ??= y;
// Roughly equivalent to x ?? (x = y)
```

请注意，与数学和按位复合赋值运算符不同，逻辑赋值运算符仅有条件地执行赋值。

请阅读[我们的解释员](https://v8.dev/features/logical-assignment)以获得更深入的解释。

## WebAssembly

### 升降在所有平台上发货

从 V8 v6.9 开始，[起飞](https://v8.dev/blog/liftoff)已被用作英特尔平台上 WebAssembly 的基准编译器（Chrome 69 在桌面系统上启用了它）。由于我们担心内存增加（因为基线编译器生成了更多的代码），到目前为止，我们将其保留在移动系统中。经过过去几个月的一些实验，我们相信在大多数情况下，内存增加可以忽略不计，因此我们最终在所有架构上默认启用Liftoff，从而提高了编译速度，特别是在arm设备（32位和64位）上。Chrome 85紧随其后，并发布了Liftoff。

### 已发货的多价值支持

WebAssembly 支持[多值代码块和函数返回](https://github.com/WebAssembly/multi-value)现已供常规使用。这反映了该提案最近在官方WebAssembly标准中的合并，并得到所有编译层的支持。

例如，这现在是一个有效的WebAssembly函数：

```wasm
(func $swap (param i32 i32) (result i32 i32)
  (local.get 1) (local.get 0)
)
```

如果该函数被导出，它也可以从 JavaScript 调用，并返回一个数组：

```js
instance.exports.swap(1, 2);
// → [2, 1]
```

相反，如果 JavaScript 函数返回一个数组（或任何迭代器），则可以将其作为 WebAssembly 模块内的多返回函数导入和调用：

```js
new WebAssembly.Instance(module, {
  imports: {
    swap: (x, y) => [y, x],
  },
});
```

```wasm
(func $main (result i32 i32)
  i32.const 0
  i32.const 1
  call $swap
)
```

更重要的是，工具链现在可以使用此功能在WebAssembly模块中生成更紧凑，更快速的代码。

### 支持 JS BigInts

WebAssembly 支持[将 WebAssembly I64 值从 JavaScript BigInts 转换为 JavaScript BigInts](https://github.com/WebAssembly/JS-BigInt-integration)已发货，可根据官方标准的最新变化进行一般使用。

因此，具有i64参数和返回值的WebAssembly函数可以从JavaScript调用而不会丢失精度：

```wasm
(module
  (func $add (param $x i64) (param $y i64) (result i64)
    local.get $x
    local.get $y
    i64.add)
  (export "add" (func $add)))
```

从 JavaScript 开始，只有 BigInts 可以作为 I64 参数传递：

```js
WebAssembly.instantiateStreaming(fetch('i64.wasm'))
  .then(({ module, instance }) => {
    instance.exports.add(12n, 30n);
    // → 42n
    instance.exports.add(12, 30);
    // → TypeError: parameters are not of type BigInt
  });
```

## V8 接口

请使用`git log branch-heads/8.4..branch-heads/8.5 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 8.5 -t branch-heads/8.5`以试验 V8 v8.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
