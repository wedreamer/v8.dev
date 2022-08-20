***

title： 'BigInt： JavaScript 中的任意精度整数'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-05-01
    标签：
*   ECMAScript
*   ES2020
*   io19
    描述： 'BigInts是JavaScript中一个新的数字基元，可以以任意精度表示整数。本文介绍了一些用例，并通过比较BigInts和JavaScript中的数字来解释Chrome 67中的新功能。
    推文：“990991035630206977”

***

`BigInt`s 是 JavaScript 中一个新的数字基元，可以表示任意精度的整数。跟`BigInt`s，您可以安全地存储和操作大整数，甚至超过安全整数限制`Number`s.本文介绍了一些用例，并通过比较来解释Chrome 67中的新功能`BigInt`s 到`Number`s 在 JavaScript 中。

## 使用案例

任意精度的整数解锁了许多 JavaScript 的新用例。

`BigInt`s 使得正确执行整数算术而不溢出成为可能。这本身就带来了无数新的可能性。例如，大数的数学运算通常用于金融技术。

[大整数 ID](https://developer.twitter.com/en/docs/basics/twitter-ids)和[高精度时间戳](https://github.com/nodejs/node/pull/20220)不能安全地表示为`Number`s 在 JavaScript 中。这[经常](https://github.com/stedolan/jq/issues/1399)导致[现实世界的错误](https://github.com/nodejs/node/issues/12115)，并导致 JavaScript 开发人员将它们表示为字符串。跟`BigInt`，则此数据现在可以表示为数值。

`BigInt`可能构成最终的基础`BigDecimal`实现。这对于以小数点精度表示货币总和并准确操作它们（又名`0.10 + 0.20 !== 0.30`问题）。

以前，具有任何这些用例的JavaScript应用程序都必须诉诸于模拟的用户空间库。`BigInt`-类似功能。什么时候`BigInt`变得广泛可用，这样的应用程序可以放弃这些运行时依赖关系，转而支持本机`BigInt`s.这有助于减少加载时间、分析时间和编译时间，最重要的是，它还提供了显著的运行时性能改进。

![The native BigInt implementation in Chrome performs better than popular userland libraries.](/\_img/bigint/performance.svg)

## 现状：`Number`{： #number }

`Number`在 JavaScript 中表示为[双精度浮子](https://en.wikipedia.org/wiki/Floating-point_arithmetic).这意味着它们的精度有限。这`Number.MAX_SAFE_INTEGER`常量给出可以安全递增的最大可能整数。其价值是`2**53-1`.

```js
const max = Number.MAX_SAFE_INTEGER;
// → 9_007_199_254_740_991
```

：：：备注
**注意：**为了便于阅读，我将每千个数字中的数字分组，使用下划线作为分隔符。[数字文本分隔符建议](/features/numeric-separators)对于常见的 JavaScript 数字文本，正是这样。
:::

递增一次会得到预期的结果：

```js
max + 1;
// → 9_007_199_254_740_992 ✅
```

但是，如果我们第二次递增它，结果就不再完全可以表示为JavaScript。`Number`:

```js
max + 2;
// → 9_007_199_254_740_992 ❌
```

注意如何`max + 1`产生的结果与`max + 2`.每当我们在JavaScript中获得这个特定的值时，就没有办法判断它是否准确。对安全整数范围之外的整数的任何计算（即`Number.MIN_SAFE_INTEGER`自`Number.MAX_SAFE_INTEGER`） 可能会丢失精度。因此，我们只能依赖安全范围内的数字整数值。

## 新热点：`BigInt`{： #bigint }

`BigInt`s 是 JavaScript 中一个新的数字基元，可以表示[任意精度](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic).跟`BigInt`s，您可以安全地存储和操作大整数，甚至超过安全整数限制`Number`s.

要创建`BigInt`，添加`n`任何整数文本的后缀。例如`123`成为`123n`.全球`BigInt(number)`函数可用于转换`Number`进入一个`BigInt`.换句话说，`BigInt(123) === 123n`.让我们使用这两种技术来解决我们之前遇到的问题：

```js
BigInt(Number.MAX_SAFE_INTEGER) + 2n;
// → 9_007_199_254_740_993n ✅
```

这是另一个例子，我们将两个相乘`Number`s:

```js
1234567890123456789 * 123;
// → 151851850485185200000 ❌
```

查看最低有效数字，`9`和`3`，我们知道乘法的结果应该以`7`（因为`9 * 3 === 27`).但是，结果以一堆零结尾。这不可能是正确的！让我们再试一次`BigInt`s 代替：

```js
1234567890123456789n * 123n;
// → 151851850485185185047n ✅
```

这次我们得到了正确的结果。

的安全整数限制`Number`不适用于`BigInt`s.因此，使用`BigInt`我们可以执行正确的整数算术，而不必担心失去精度。

### 新的原语

`BigInt`s 是 JavaScript 语言中的新原语。因此，他们得到了自己的类型，可以使用`typeof`算子：

```js
typeof 123;
// → 'number'
typeof 123n;
// → 'bigint'
```

因为`BigInt`s 是一个单独的类型，一个`BigInt`从不严格等于`Number`，例如`42n !== 42`.比较`BigInt`到`Number`，在进行比较之前将其中一个转换为另一个的类型或使用抽象相等（`==`):

```js
42n === BigInt(42);
// → true
42n == 42;
// → true
```

当强制转换为布尔值时（当使用`if`,`&&`,`||`或`Boolean(int)`，例如），`BigInt`遵循与`Number`s.

```js
if (0n) {
  console.log('if');
} else {
  console.log('else');
}
// → logs 'else', because `0n` is falsy.
```

### 运营商

`BigInt`s 支持最常见的运算符。二元的`+`,`-`,`*`和`**`一切都按预期工作。`/`和`%`工作，并根据需要向零舍入。按位运算`|`,`&`,`<<`,`>>`和`^`执行按位算术，假设[二的补码表示](https://en.wikipedia.org/wiki/Two%27s_complement)对于负值，就像它们对负值所做的那样`Number`s.

```js
(7 + 6 - 5) * 4 ** 3 / 2 % 3;
// → 1
(7n + 6n - 5n) * 4n ** 3n / 2n % 3n;
// → 1n
```

元`-`可用于表示负数`BigInt`值，例如`-42n`.元`+`是*不*支持，因为它会破坏asm.js期望的代码`+x`始终生成`Number`或例外。

一个诀窍是不允许混合操作`BigInt`s 和`Number`s.这是一件好事，因为任何隐含的胁迫都可能丢失信息。请考虑以下示例：

```js
BigInt(Number.MAX_SAFE_INTEGER) + 2.5;
// → ?? 🤔
```

结果应该是什么？这里没有很好的答案。`BigInt`s 不能表示分数，并且`Number`s 不能表示`BigInt`s 超出安全整数限制。因此，混合操作之间`BigInt`s 和`Number`s 的结果`TypeError`例外。

此规则的唯一例外是比较运算符，例如`===`（如前所述），`<`和`>=`– 因为它们返回布尔值，所以没有精度损失的风险。

```js
1 + 1n;
// → TypeError
123 < 124n;
// → true
```

因为`BigInt`s 和`Number`s一般不要混合，请避免重载或神奇地“升级”您现有的代码来使用`BigInt`s 而不是`Number`s.决定在这两个域中哪一个进行操作，然后坚持下去。为*新增功能*对潜在大整数进行操作的 API，`BigInt`是最好的选择。`Number`对于已知在安全整数范围内的整数值，s 仍然有意义。

另一件需要注意的事情是[这`>>>`算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators#Unsigned_right_shift)，它执行无符号右移，对于`BigInt`，因为它们总是被签名。因此，`>>>`不适用于`BigInt`s.

### 应用程序接口

几个新的`BigInt`- 提供特定的API。

全球`BigInt`构造函数类似于`Number`构造函数：它将其参数转换为`BigInt`（如前所述）。如果转换失败，则会抛出一个`SyntaxError`或`RangeError`例外。

```js
BigInt(123);
// → 123n
BigInt(1.5);
// → RangeError
BigInt('1.5');
// → SyntaxError
```

这些示例中的第一个示例将数字文本传递给`BigInt()`.这是一种不好的做法，因为`Number`s 遭受精度损失，因此我们可能已经在`BigInt`转化发生：

```js
BigInt(123456789123456789);
// → 123456789123456784n ❌
```

因此，我们建议坚持使用`BigInt`文字表示法（使用`n`后缀），或传递字符串（不是`Number`!)自`BigInt()`相反：

```js
123456789123456789n;
// → 123456789123456789n ✅
BigInt('123456789123456789');
// → 123456789123456789n ✅
```

两个库函数支持包装`BigInt`值为有符号或无符号整数，限制为特定位数。`BigInt.asIntN(width, value)`包装一个`BigInt`值`width`-数字二进制有符号整数，以及`BigInt.asUintN(width, value)`包装一个`BigInt`值`width`-数字二进制无符号整数。例如，如果您正在执行 64 位算术运算，则可以使用以下 API 保持在适当的范围内：

```js
// Highest possible BigInt value that can be represented as a
// signed 64-bit integer.
const max = 2n ** (64n - 1n) - 1n;
BigInt.asIntN(64, max);
→ 9223372036854775807n
BigInt.asIntN(64, max + 1n);
// → -9223372036854775808n
//   ^ negative because of overflow
```

请注意，一旦我们通过`BigInt`值超过 64 位整数范围（即绝对数值为 63 位 + 符号为 1 位）。

`BigInt`s 可以准确地表示 64 位有符号和无符号整数，这些整数通常用于其他编程语言。两种新的类型化数组风格，`BigInt64Array`和`BigUint64Array`，可以更轻松地有效地表示和操作此类值的列表：

```js
const view = new BigInt64Array(4);
// → [0n, 0n, 0n, 0n]
view.length;
// → 4
view[0];
// → 0n
view[0] = 42n;
view[0];
// → 42n
```

这`BigInt64Array`flavor 确保其值保持在有符号的 64 位限制内。

```js
// Highest possible BigInt value that can be represented as a
// signed 64-bit integer.
const max = 2n ** (64n - 1n) - 1n;
view[0] = max;
view[0];
// → 9_223_372_036_854_775_807n
view[0] = max + 1n;
view[0];
// → -9_223_372_036_854_775_808n
//   ^ negative because of overflow
```

这`BigUint64Array`flavor 使用无符号的 64 位限制来执行相同的操作。

## Polyfilling and transpiling BigInts { #polyfilling-transpiling }

在撰写本文时，`BigInt`s 仅在 Chrome 中受支持。其他浏览器正在积极努力实现它们。但是，如果您想使用，该怎么办`BigInt`功能性*今天*而不牺牲浏览器兼容性？我很高兴你问！答案是...至少可以说，很有趣。

与大多数其他现代JavaScript功能不同，`BigInt`s 不能合理地转译为 ES5。

这`BigInt`建议[更改操作员的行为](#operators)（如`+`,`>=`等）要处理`BigInt`s.这些变化不可能直接进行polyfill，并且它们也使得（在大多数情况下）转码是不可行的。`BigInt`代码以使用 Babel 或类似工具回退代码。原因是这样的转译必须取代*每个操作员*在程序中调用某个函数，该函数对其输入执行类型检查，这将导致不可接受的运行时性能损失。此外，它还会大大增加任何转译捆绑包的文件大小，从而对下载、解析和编译时间产生负面影响。

一个更可行和面向未来的解决方案是使用[JSBI 库](https://github.com/GoogleChromeLabs/jsbi#why)现在。JSBI 是`BigInt`V8 和 Chrome 中的实现 — 根据设计，它的行为与本机完全相同`BigInt`功能性。不同之处在于，它不是依赖于语法，而是公开了[一个接口](https://github.com/GoogleChromeLabs/jsbi#how):

```js
import JSBI from './jsbi.mjs';

const max = JSBI.BigInt(Number.MAX_SAFE_INTEGER);
const two = JSBI.BigInt('2');
const result = JSBI.add(max, two);
console.log(result.toString());
// → '9007199254740993'
```

一次`BigInt`s 在您关心的所有浏览器中都得到本机支持，您可以[用`babel-plugin-transform-jsbi-to-bigint`将代码转译为本机代码`BigInt`法典](https://github.com/GoogleChromeLabs/babel-plugin-transform-jsbi-to-bigint)并删除 JSBI 依赖项。例如，上面的示例将转译为：

```js
const max = BigInt(Number.MAX_SAFE_INTEGER);
const two = 2n;
const result = max + two;
console.log(result);
// → '9007199254740993'
```

## 进一步阅读

如果您对如何`BigInt`在幕后工作（例如，它们如何在内存中表示，以及如何对它们执行操作），[阅读我们的 V8 博客文章，了解实施详情](/blog/bigint).

## `BigInt`支持 { #support }

<feature-support chrome="67 /blog/bigint"
              firefox="68 https://wingolog.org/archives/2019/05/23/bigint-shipping-in-firefox"
              safari="14"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes #polyfilling-transpiling"></feature-support>
