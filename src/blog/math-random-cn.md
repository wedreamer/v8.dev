***

标题： '有`Math.random()`，然后有`Math.random()`'
作者： '杨果 （[@hashseed](https://twitter.com/hashseed)），软件工程师和骰子设计师
化身：

*   “阳国”
    日期： 2015-12-17 13：33：37
    标签：
*   ECMAScript
*   内部
    描述： 'V8 的 Math.random 实现现在使用一种名为 xorshift128+ 的算法，与旧的 MWC1616 实现相比，提高了随机性。

***

> `Math.random()`返回`Number`带正号的值，大于或等于`0`但小于`1`，使用依赖于实现的算法或策略，随机或伪随机选择，在该范围内具有近似均匀的分布。此函数不带任何参数。

—*[ES 2015，第 20.2.2.27 节](http://tc39.es/ecma262/#sec-math.random)*

`Math.random()`是Javascript中最知名和最常用的随机性来源。在 V8 和大多数其他 Javascript 引擎中，它是使用[伪随机数生成器](https://en.wikipedia.org/wiki/Pseudorandom_number_generator)（PRNG）。与所有PRNG一样，随机数是从内部状态派生的，每个新随机数的固定算法都会改变该状态。因此，对于给定的初始状态，随机数序列是确定性的。由于内部状态的位大小 n 是有限的，因此 PRNG 生成的数字最终将重复。此周期长度的上限[排列周期](https://en.wikipedia.org/wiki/Cyclic_permutation)为 2<sup>n</sup>.

有许多不同的PRNG算法;其中最知名的是[梅森-费尔托斯特](https://en.wikipedia.org/wiki/Mersenne_Twister)和[断续器](https://en.wikipedia.org/wiki/Linear_congruential_generator).每种都有其特定的特征，优点和缺点。理想情况下，它将为初始状态使用尽可能少的内存，快速执行，具有较大的周期长度，并提供高质量的随机分布。虽然可以轻松测量或计算内存使用情况、性能和周期长度，但质量更难确定。统计测试背后有很多数学来检查随机数的质量。事实上的标准PRNG测试套件，[测试U01](http://simul.iro.umontreal.ca/testu01/tu01.html)，则实现其中的许多测试。

直到[最近](https://github.com/v8/v8/blob/ceade6cf239e0773213d53d55c36b19231c820b5/src/js/math.js#L143)（直到4.9.40版本），V8选择PRNG是MWC1616（乘以carry，结合两个16位部分）。它使用64位的内部状态，大致如下所示：

```cpp
uint32_t state0 = 1;
uint32_t state1 = 2;
uint32_t mwc1616() {
  state0 = 18030 * (state0 & 0xFFFF) + (state0 >> 16);
  state1 = 30903 * (state1 & 0xFFFF) + (state1 >> 16);
  return state0 << 16 + (state1 & 0xFFFF);
}
```

然后，32 位值转换为与规范一致的 0 到 1 之间的浮点数。

MWC1616使用很少的内存，计算速度非常快，但不幸的是，它的质量低于标准：

*   它可以生成的随机值的数量限制为 2<sup>32</sup>与 2 相反<sup>52</sup>双精度浮点可以表示的介于 0 和 1 之间的数字。
*   结果中更重要的上半部分几乎完全取决于 state0 的值。周期长度最多为 2<sup>32</sup>，但不是几个大的排列周期，而是有许多短的排列周期。如果初始状态选择不当，周期长度可能小于4000万。
*   它未能通过 TestU01 套件中的许多统计测试。

这已经[指出](https://medium.com/@betable/tifu-by-using-math-random-f1c308c4fd9d)对我们来说，在了解了问题并经过一些研究之后，我们决定重新实现`Math.random`基于一种算法，称为[异速128+](http://vigna.di.unimi.it/ftp/papers/xorshiftplus.pdf).它使用128位内部状态，周期长度为2<sup>128</sup>- 1，并通过TestU01套件的所有测试。

实现[登陆 V8 v4.9.41.0](https://github.com/v8/v8/blob/085fed0fb5c3b0136827b5d7c190b4bd1c23a23e/src/base/utils/random-number-generator.h#L102)在我们意识到这个问题的几天内。它将在Chrome 49中可用。双[火狐浏览器](https://bugzilla.mozilla.org/show_bug.cgi?id=322529#c99)和[野生动物园](https://bugs.webkit.org/show_bug.cgi?id=151641)也切换到xorshift128 +。

在 V8 7.1 中，再次调整了含义[华氯](https://chromium-review.googlesource.com/c/v8/v8/+/1238551/5)仅在 state0 上中继。请在[源代码](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/base/utils/random-number-generator.h;l=119?q=XorShift128\&sq=\&ss=chromium).

但是，不要搞错：尽管xorshift128 +是MWC1616的巨大改进，但它仍然不是[加密安全](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator).对于散列、签名生成、加密/解密等用例，普通 PRNG 是不合适的。Web 加密 API 引入了[`window.crypto.getRandomValues`](https://developer.mozilla.org/en-US/docs/Web/API/RandomSource/getRandomValues)，这是一种以性能为代价返回加密安全随机值的方法。

请记住，如果您在 V8 和 Chrome 中发现需要改进的领域，即使是像这样不会直接影响规范合规性、稳定性或安全性的领域，请提交[我们的错误跟踪器上的问题](https://bugs.chromium.org/p/v8/issues/entry?template=Defect%20report%20from%20user).
