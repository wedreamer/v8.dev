***

标题： “懒惰实习：去优化函数的懒惰解链接”
作者： '朱莉安娜·佛朗哥 （[@jupvfranco](https://twitter.com/jupvfranco)），懒惰专家'
日期： 2017-10-04 13：33：37
标签：

*   记忆
*   内部
    描述： “此技术深入探讨了 V8 过去如何取消链接已取消优化的函数，以及我们最近如何对其进行更改以提高性能。
    推文：“915473224187760640”

***

大约三个月前，我以实习生的身份加入了V8团队（谷歌慕尼黑），从那时起，我一直在研究VM的*去优化器*- 对我来说是全新的，被证明是一个有趣且具有挑战性的项目。我实习的第一部分集中在[在安全性方面提高 VM 的安全性](https://docs.google.com/document/d/1ELgd71B6iBaU6UmZ_lvwxf_OrYYnv0e4nuzZpK05-pg/edit).第二部分侧重于性能改进。也就是说，删除用于取消以前取消优化的函数链接的数据结构，这是垃圾回收期间的性能瓶颈。这篇博客文章描述了我实习的第二部分。我将解释 V8 过去如何取消链接已取消优化的函数，我们如何更改此函数，以及获得了哪些性能改进。

让我们（非常）简要地回顾一下JavaScript函数的V8管道：V8的解释器Ignition在解释该函数时收集有关该函数的分析信息。一旦函数变热，这些信息就会传递给V8的编译器TurboFan，后者会生成优化的机器代码。当分析信息不再有效时（例如，由于其中一个分析对象在运行时获得不同的类型），优化的机器代码可能会变得无效。在这种情况下，V8 需要对其进行去优化。

![An overview of V8, as seen in JavaScript Start-up Performance](../_img/lazy-unlinking/v8-overview.png)

优化后，TurboFan会为优化中的功能生成一个代码对象，即优化的机器代码。下次调用此函数时，V8 将按照该函数的优化代码链接执行它。取消优化此函数后，我们需要取消链接代码对象，以确保它不会再次执行。这是怎么发生的呢？

例如，在下面的代码中，函数`f1`将被多次调用（始终将整数作为参数传递）。然后，TurboFan会为该特定情况生成机器代码。

```js
function g() {
  return (i) => i;
}

// Create a closure.
const f1 = g();
// Optimize f1.
for (var i = 0; i < 1000; i++) f1(0);
```

每个函数还为解释器提供蹦床 - 更多细节在这些[幻灯片](https://docs.google.com/presentation/d/1Z6oCocRASCfTqGq1GCo1jbULDGS-w-nzxkbVF7Up0u0/edit#slide=id.p)— 并将在其中保留指向此蹦床的指针`SharedFunctionInfo`（SFI）。每当V8需要返回未优化的代码时，都会使用此蹦床。因此，在通过传递不同类型的参数触发的取消优化时，例如，Deoptimizer可以简单地将JavaScript函数的代码字段设置为此蹦床。

![An overview of V8, as seen in JavaScript Start-up Performance](../_img/lazy-unlinking/v8-overview.png)

虽然这看起来很简单，但它迫使 V8 保留优化的 JavaScript 函数的弱列表。这是因为可以有不同的函数指向同一个优化的代码对象。我们可以按如下方式扩展我们的示例，以及函数`f1`和`f2`两者都指向相同的优化代码。

```js
const f2 = g();
f2(0);
```

如果函数`f1`已取消优化（例如，通过使用不同类型的对象调用它`{x: 0}`）我们需要确保无效的代码不会通过调用再次执行`f2`.

因此，在去优化时，V8 用于迭代所有优化的 JavaScript 函数，并将取消链接那些指向代码对象被解优化的函数。在具有许多优化的JavaScript函数的应用程序中的这种迭代成为性能瓶颈。此外，除了减慢去优化速度之外，V8 还习惯于在停止垃圾回收的循环时迭代这些列表，使情况变得更糟。

为了了解这种数据结构对V8性能的影响，我们写了一个[微基准](https://github.com/v8/v8/blob/master/test/js-perf-test/ManyClosures/create-many-closures.js)通过在创建许多JavaScript函数后触发许多清除周期来强调其使用。

```js
function g() {
  return (i) => i + 1;
}

// Create an initial closure and optimize.
var f = g();

f(0);
f(0);
%OptimizeFunctionOnNextCall(f);
f(0);

// Create 2M closures; those will get the previously optimized code.
var a = [];
for (var i = 0; i < 2000000; i++) {
  var h = g();
  h();
  a.push(h);
}

// Now cause scavenges; all of them are slow.
for (var i = 0; i < 1000; i++) {
  new Array(50000);
}
```

在运行此基准测试时，我们可以观察到 V8 在垃圾回收上花费了大约 98% 的执行时间。然后，我们删除了此数据结构，而是使用了一种方法*惰性取消链接*，这是我们在 x64 上观察到的：

![](../_img/lazy-unlinking/microbenchmark-results.png)

虽然这只是一个创建许多JavaScript函数并触发许多垃圾回收周期的微基准测试，但它让我们了解了这种数据结构引入的开销。其他更现实的应用程序，我们看到一些开销，并激励这项工作，是[路由器基准测试](https://github.com/delvedor/router-benchmark)在 Node 中实现.js和[ARES-6 基准套件](http://browserbench.org/ARES-6/).

## 延迟取消链接

V8 不是在解优化时将优化的代码与 JavaScript 函数取消链接，而是将其推迟到下次调用此类函数时。当调用这些函数时，V8 会检查它们是否已被取消优化，取消链接它们，然后继续它们的惰性编译。如果再也不调用这些函数，则永远不会取消链接它们，也不会收集已取消优化的代码对象。但是，假设在取消优化期间，我们会使代码对象的所有嵌入字段失效，因此我们只使该代码对象保持活动状态。

这[犯](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690)删除了此优化的 JavaScript 函数列表，需要在 VM 的几个部分中进行更改，但基本思路如下。在组装优化的代码对象时，我们检查这是否是JavaScript函数的代码。如果是这样，在其序言中，我们组装机器代码以在代码对象已取消优化时进行救援。在取消优化时，我们不会修改取消优化的代码 - 代码修补已经消失。因此，它的位`marked_for_deoptimization`再次调用函数时仍设置。TurboFan生成代码来检查它，如果它被设置，那么V8跳转到一个新的内置，`CompileLazyDeoptimizedCode`，这将取消已取消优化代码与 JavaScript 函数的链接，然后继续进行惰性编译。

更详细地说，第一步是生成加载当前正在组装的代码的地址的指令。我们可以在x64中使用以下代码执行此操作：

```cpp
Label current;
// Load effective address of current instruction into rcx.
__ leaq(rcx, Operand(&current));
__ bind(&current);
```

之后，我们需要获取代码对象中的位置`marked_for_deoptimization`位生活。

```cpp
int pc = __ pc_offset();
int offset = Code::kKindSpecificFlags1Offset - (Code::kHeaderSize + pc);
```

然后我们可以测试该位，如果它被设置，我们跳转到`CompileLazyDeoptimizedCode`内置。

```cpp
// Test if the bit is set, that is, if the code is marked for deoptimization.
__ testl(Operand(rcx, offset),
         Immediate(1 << Code::kMarkedForDeoptimizationBit));
// Jump to builtin if it is.
__ j(not_zero, /* handle to builtin code here */, RelocInfo::CODE_TARGET);
```

在这一边`CompileLazyDeoptimizedCode`内置，剩下要做的就是从JavaScript函数中取消链接代码字段，并将其设置为蹦床到解释器条目。因此，考虑到JavaScript函数的地址在寄存器中`rdi`，我们可以获得指向`SharedFunctionInfo`跟：

```cpp
// Field read to obtain the SharedFunctionInfo.
__ movq(rcx, FieldOperand(rdi, JSFunction::kSharedFunctionInfoOffset));
```

...同样，蹦床具有：

```cpp
// Field read to obtain the code object.
__ movq(rcx, FieldOperand(rcx, SharedFunctionInfo::kCodeOffset));
```

然后，我们可以使用它来更新代码指针的函数槽：

```cpp
// Update the code field of the function with the trampoline.
__ movq(FieldOperand(rdi, JSFunction::kCodeOffset), rcx);
// Write barrier to protect the field.
__ RecordWriteField(rdi, JSFunction::kCodeOffset, rcx, r15,
                    kDontSaveFPRegs, OMIT_REMEMBERED_SET, OMIT_SMI_CHECK);
```

这将产生与以前相同的结果。但是，与其在解优化程序中处理取消链接，不如在代码生成期间担心它。因此，手写组件。

以上是[它在 x64 体系结构中的工作原理](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-0920a0f56f95b36cdd43120466ec7ccd).我们已经实施了它[ia32](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-10985b50f31627688e9399a768d9ec21),[手臂](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-0f5515e80dd0139244a4ae48ce56a139),[手臂64](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-1bbe32f45000ec9157f4997a6c95f1b1),[米普斯](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-73f690ee13a5465909ae9fc1a70d8c41)和[mips64](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690#diff-b1de25cbfd2d02b81962797bfdf807df)也。

这种新技术已经集成到 V8 中，正如我们稍后将讨论的那样，它允许性能改进。但是，它有一个小缺点：以前，V8 只会考虑在取消优化时才取消链接。现在，它必须在激活所有优化功能时这样做。此外，检查的方法`marked_for_deoptimization`bit 并不像它可能的那样有效，因为我们需要做一些工作来获取代码对象的地址。请注意，在输入每个优化函数时会发生这种情况。此问题的可能解决方案是在代码对象中保留指向自身的指针。V8 不是在每次调用函数时都做一次查找代码对象地址的工作，而是在构造函数后只执行一次。

## 结果

现在，我们来看看这个项目获得的性能提升和回归。

### x64 上的常规改进

下图向我们展示了相对于上一次提交而言的一些改进和回归。请注意，越高越好。

![](../_img/lazy-unlinking/x64.png)

这`promises`基准测试是我们看到更大改进的基准测试，观察到`bluebird-parallel`基准，和 22.40%`wikipedia`.我们还在一些基准测试中观察到一些回归。这与上面解释的问题有关，即检查代码是否标记为取消优化。

我们还看到了ARES-6基准测试套件的改进。请注意，在此图表中，也越高越好。这些计划过去在与GC相关的活动中花费了大量时间。通过延迟取消链接，我们总体上将性能提高了1.9%。最值得注意的案例是`Air steadyState`我们提高了约5.36%。

![](../_img/lazy-unlinking/ares6.png)

### 我们之前的结果

Octane和ARES-6基准套件的性能结果也显示在AreWeFastYet跟踪器上。我们在 2017 年 9 月 5 日使用提供的默认计算机（macOS 10.10 64 位、Mac Pro、shell）查看了这些性能结果。

![Cross-browser results on Octane as seen on AreWeFastYet](../_img/lazy-unlinking/awfy-octane.png)

![Cross-browser results on ARES-6 as seen on AreWeFastYet](../_img/lazy-unlinking/awfy-ares6.png)

### 对节点的影响.js

我们还可以看到性能改进`router-benchmark`.以下两个图显示了每个测试路由器每秒的操作数。因此，越高越好。我们已经用这个基准测试套件进行了两种实验。首先，我们单独运行每个测试，以便我们可以独立于其余测试看到性能改进。其次，我们一次运行所有测试，无需切换 VM，从而模拟了每个测试都与其他功能集成的环境。

对于第一个实验，我们看到`router`和`express`测试在相同的时间内执行的操作次数大约是以前两倍。对于第二个实验，我们看到了更大的改进。在某些情况下，例如`routr`,`server-router`和`router`，基准测试分别执行大约 3.80×、3× 和 2×以上的操作。发生这种情况是因为 V8 积累了更多优化的 JavaScript 函数，一次又一次地进行测试。因此，每当执行给定的测试时，如果触发了垃圾回收周期，V8 必须访问当前测试和先前测试中的优化函数。

![](../_img/lazy-unlinking/router.png)

![](../_img/lazy-unlinking/router-integrated.png)

### 进一步优化

现在 V8 没有在上下文中保留 JavaScript 函数的链接列表，我们可以删除该字段`next`从`JSFunction`类。尽管这是一个简单的修改，但它允许我们保存每个函数的指针大小，这代表了在几个网页中显着节省：

：：：表包装器
|基准|种类|内存节省（绝对）|内存节省（相对）|
|------------ |--------------------------------- |------------------------- |------------------------- |
|facebook.com |平均有效量|170 千字节 |3.70% |
|twitter.com |已分配对象的平均大小|284 千字节 |1.20% |
|cnn.com |已分配对象的平均大小|788 千字节 |1.53% |
|youtube.com |已分配对象的平均大小|129 千字节 |0.79% |
:::

## 确认

在我的整个实习期间，我得到了几个人的很多帮助，他们总是可以回答我的许多问题。因此，我要感谢以下人员：Benedikt Meurer，Jaroslav Sevcik和Michael Starzinger关于编译器和去优化器如何工作的讨论，Ulan Degenbaev在我破坏垃圾收集器时帮助垃圾收集器，Mathias Bynens，Peter Marshall，Camillo Bruni和Maya Armyanova校对本文。

最后，这篇文章是我作为Google实习生的最后一次贡献，我想借此机会感谢V8团队中的每个人，特别是我的主持人Benedikt Meurer，感谢他们接待我，并让我有机会参与这样一个有趣的项目 - 我绝对学到了很多东西，很享受我在Google的时光！
