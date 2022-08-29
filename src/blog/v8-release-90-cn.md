***

标题： 'V8 发布 v9.0'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser)），直排在
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-03-17
    标签：
*   释放
    描述： 'V8 版本 v9.0 带来了对 RegExp 匹配索引的支持和各种性能改进。
    推文：“1372227274712494084”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.0](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.0)，直到几周后与Chrome 90 Stable合作发布之前，它一直处于测试阶段。V8 v9.0 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 正则表达式匹配索引

从 v9.0 开始，开发人员可以选择在正则表达式匹配中获取匹配捕获组的开始和结束位置数组。此阵列可通过`.indices`属性，当正则表达式具有`/d`旗。

```javascript
const re = /(a)(b)/d;      // Note the /d flag.
const m = re.exec('ab');
console.log(m.indices[0]); // Index 0 is the whole match.
// → [0, 2]
console.log(m.indices[1]); // Index 1 is the 1st capture group.
// → [0, 1]
console.log(m.indices[2]); // Index 2 is the 2nd capture group.
// → [1, 2]
```

请参阅[我们的解释员](https://v8.dev/features/regexp-match-indices)进行深入潜水。

### 更快`super`属性访问

访问`super`属性（例如，`super.x`） 通过使用 V8 的内联缓存系统和 TurboFan 中优化的代码生成进行了优化。随着这些变化，`super`属性访问现在更接近于常规属性访问，如下图所示。

![Compare super property access to regular property access, optimized](../_img/fast-super/super-opt.svg)

请参阅[专门的博客文章](https://v8.dev/blog/fast-super)了解更多详情。

### `for ( async of`禁止

一个[语法歧义](https://github.com/tc39/ecma262/issues/2034)最近被发现和[固定](https://chromium-review.googlesource.com/c/v8/v8/+/2683221)在 V8 v9.0 中。

令牌序列`for ( async of`现在不再解析。

## WebAssembly

### 更快的 JS 到 Wasm 调用

V8 对 WebAssembly 和 JavaScript 函数的参数使用不同的表示形式。出于这个原因，当JavaScript调用导出的WebAssembly函数时，调用会通过一个所谓的*JS-to-Wasm wrapper*，负责将参数从 JavaScript 土地调整到 WebAssembly 土地，以及将结果调整到相反的方向。

不幸的是，这带来了性能成本，这意味着从JavaScript到WebAssembly的调用不如从JavaScript到JavaScript的调用快。为了最大限度地减少这种开销，现在可以在调用站点内联JS到Wasm包装器，从而简化代码并删除此额外的帧。

假设我们有一个WebAssembly函数来添加两个双浮点数，如下所示：

```cpp
double addNumbers(double x, double y) {
  return x + y;
}
```

假设我们从JavaScript调用它来添加一些向量（表示为类型化数组）：

```javascript
const addNumbers = instance.exports.addNumbers;

function vectorSum(len, v1, v2) {
  const result = new Float64Array(len);
  for (let i = 0; i < len; i++) {
    result[i] = addNumbers(v1[i], v2[i]);
  }
  return result;
}

const N = 100_000_000;
const v1 = new Float64Array(N);
const v2 = new Float64Array(N);
for (let i = 0; i < N; i++) {
  v1[i] = Math.random();
  v2[i] = Math.random();
}

// Warm up.
for (let i = 0; i < 5; i++) {
  vectorSum(N, v1, v2);
}

// Measure.
console.time();
const result = vectorSum(N, v1, v2);
console.timeEnd();
```

在这个简化的微基准上，我们看到了以下改进：

![Microbenchmark comparison](../_img/v8-release-90/js-to-wasm.svg)

该功能仍处于实验阶段，可以通过`--turbo-inline-js-wasm-calls`旗。

有关更多详细信息，请参阅[设计文档](https://docs.google.com/document/d/1mXxYnYN77tK-R1JOVo6tFG3jNpMzfueQN1Zp5h3r9aM/edit).

## V8 接口

请使用`git log branch-heads/8.9..branch-heads/9.0 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.0 -t branch-heads/9.0`以试验 V8 v9.0 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
