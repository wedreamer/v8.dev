***

title： 'Emscripten and the LLVM WebAssembly backend'
作者： '阿隆扎凯'
化身：

*   'alon-zakai'
    日： 2019-07-01 16：45：00
    标签：
*   WebAssembly
*   工具
    描述： 'Emscripten正在切换到LLVM WebAssembly后端，从而缩短了链接时间s️并带来了许多其他好处。
    推文：“1145704863377981445”

***

WebAssembly通常是从源语言编译的，这意味着开发人员需要*工具*以使用它。正因为如此，V8团队致力于相关的开源项目，例如[LLVM](http://llvm.org/),[Emscripten](https://emscripten.org/),[二进制](https://github.com/WebAssembly/binaryen/)和[瓦布特](https://github.com/WebAssembly/wabt).这篇文章描述了我们在Emscripten和LLVM上所做的一些工作，这些工作将很快允许Emscripten切换到[LLVM WebAssembly backend](https://github.com/llvm/llvm-project/tree/master/llvm/lib/Target/WebAssembly)默认情况下 - 请对其进行测试并报告任何问题！

LLVM WebAssembly后端在Emscripten中已经成为一个选项已经有一段时间了，因为我们一直在后端上工作，与它在Emscripten中的集成并行，并与开源WebAssembly工具社区中的其他人合作。它现在已经达到了WebAssembly后端击败旧的”[快速复合](https://github.com/emscripten-core/emscripten-fastcomp/)“上的后端，因此我们希望将默认值切换到它。这个公告是在此之前发生的，以便首先进行尽可能多的测试。

这是一个重要的升级，有几个令人兴奋的原因：

*   **链接速度更快**：LLVM WebAssembly 后端与[`wasm-ld`](https://lld.llvm.org/WebAssembly.html)完全支持使用 WebAssembly 对象文件的增量编译。Fastcomp在位码文件中使用了LLVM IR，这意味着在链接时所有IR将由LLVM编译。这是链接时间慢的主要原因。另一方面，使用WebAssembly对象文件，`.o`文件包含已经编译的WebAssembly（以可以链接的可重定位形式，很像原生链接）。因此，链接步骤可能比fastcomp快得多 - 我们将在下面看到一个真实世界的测量，加速速度为7×！
*   **更快、更小的代码**：我们在LLVM WebAssembly后端以及Emscripten之后运行的Binaryen优化器上努力工作。结果是，在我们跟踪的大多数基准测试中，LLVM WebAssembly后端路径现在在速度和大小上都超过了fastcomp。
*   **支持所有 LLVM IR**：Fastcomp 可以处理由`clang`，但由于其体系结构，它经常在其他来源上失败，特别是在将IR“合法化”为fastcomp可以处理的类型方面。另一方面，LLVM WebAssembly后端使用通用的LLVM后端基础架构，因此它可以处理所有内容。
*   **新的 WebAssembly 功能**：Fastcomp 在运行前编译为 asm.js`asm2wasm`，这意味着很难处理新的 WebAssembly 功能，如尾部调用、异常、SIMD 等。WebAssembly后端是处理这些功能的自然场所，实际上我们正在处理刚才提到的所有功能！
*   **从上游进行更快的常规更新**：与最后一点相关，使用上游WebAssembly后端意味着我们可以随时使用最新的LLVM上游，这意味着我们可以 C++在`clang`，新的LLVM IR优化等，一旦落地。

## 测试

要测试 WebAssembly 后端，只需使用[最近的`emsdk`](https://github.com/emscripten-core/emsdk)并执行

    emsdk install latest-upstream
    emsdk activate latest-upstream

这里的“上游”是指LLVM WebAssembly后端位于上游LLVM中，这与fastcomp不同。事实上，由于它位于上游，因此您不需要使用`emsdk`如果你构建普通的LLVM+`clang`你自己！（要将这样的构建与Emscripten一起使用，只需在`.emscripten`文件。

当前使用`emsdk [install|activate] latest`仍然使用 fastcomp。还有“latest-fastcomp”，它做同样的事情。当我们切换默认后端时，我们会让“latest”和“latest-upstream”做同样的事情，到时候“latest-fastcomp”将是获得fastcomp的唯一途径。Fastcomp仍然是一个选项，而它仍然有用;请参阅最后有关此内容的更多注释。

## 历史

这将是**第三**Emscripten 中的后端，以及**第二**迁移。第一个后端是用JavaScript编写的，并以文本形式解析LLVM IR。这在2010年的实验中很有用，但有明显的缺点，包括LLVM的文本格式会发生变化，编译速度没有我们想要的那么快。2013年，一个新的后端被写入LLVM的一个分支中，绰号“fastcomp”。它被设计为发射[asm.js](https://en.wikipedia.org/wiki/Asm.js)，早期的JS后端已被黑客入侵（但做得不是很好）。因此，它在代码质量和编译时间方面有了很大的提高。

这也是Emscripten中相对较小的变化。虽然Emscripten是一个编译器，但原始的后端和fastcomp一直是项目中相当小的一部分 - 更多的代码进入系统库，工具链集成，语言绑定等。因此，虽然切换编译器后端是一个戏剧性的变化，但它只影响整个项目的一部分。

## 基准

### 代码大小

![Code size measurements (lower is better)](/\_img/emscripten-llvm-wasm/size.svg)

（此处的所有尺寸都标准化为 fastcomp。如您所见，WebAssembly后端的大小几乎总是更小！这种差异在左侧较小的微基准标记（小写名称）上更为明显，其中系统库的新改进更重要。但是，即使在右侧的大多数宏模板标记（大写名称）上，代码大小也会减小，这些宏模板是现实世界的代码库。宏基准上的一个回归是LZMA，其中较新的LLVM做出不同的内联决策，最终不走运。

总体而言，宏观桥标记平均收缩**3.7%**.对于编译器升级来说还不错！例如，我们在测试套件中没有的实际代码库上看到类似的东西，[香蕉面包](https://github.com/kripken/BananaBread/)，的端口[立方体 2 游戏引擎](http://cubeengine.com/)到网络，缩小超过**6%**和[《毁灭战士3》缩小](http://www.continuation-labs.com/projects/d3wasm/) **15%**!

这些大小改进（以及我们接下来将讨论的速度改进）是由于以下几个因素造成的：

*   LLVM的后端代码是智能的，可以做像fastcomp这样的简单后端无法做到的事情，比如[断续器](https://en.wikipedia.org/wiki/Value_numbering).
*   较新的LLVM具有更好的IR优化。
*   如前所述，我们在WebAssembly后端的输出上调整Binaryen优化器方面做了很多工作。

### 速度

![Speed measurements (lower is better)](/\_img/emscripten-llvm-wasm/speed.svg)

（测量值在 V8 上。在微基准标记中，速度是一幅喜忧参半的画面——这并不奇怪，因为它们中的大多数都由单个函数甚至循环主导，因此对Emscripten发出的代码的任何更改都可能导致VM做出幸运或不幸的优化选择。总体而言，大约相同数量的微基准与改善或倒退的微基准保持不变。看看更现实的宏观模板标记，LZMA再次成为异常值，再次因为前面提到的一个不吉利的内联决策，但除此之外，每个宏观模板标记都会得到改善！

宏基准上的平均变化是**3.2%**.

### 构建时间

![Compile and link time measurements on BananaBread (lower is better)](/\_img/emscripten-llvm-wasm/build.svg)

构建时间的变化会因项目而异，但这里有一些来自BananaBread的示例数字，这是一个完整但紧凑的游戏引擎，由112个文件和95，287行代码组成。在左侧，我们有编译步骤的构建时间，即使用项目的默认步骤将源文件编译为对象文件`-O3`（所有时间都规范化为 fastcomp）。正如你所看到的，WebAssembly后端的编译步骤需要稍长的时间，这是有道理的，因为我们在这个阶段正在做更多的工作 - 而不是像fastcomp那样只是编译源代码到位码，我们还将比特码编译为WebAssembly。

在右侧，我们有链接步骤的数字（也规范化为fastcomp），即生成最终的可执行文件，这里带有`-O0`这适用于增量构建（对于完全优化的构建，您可能会使用`-O3`以及，见下文）。事实证明，编译步骤中的轻微增加是值得的，因为链接是**快 7× 以上**!这就是增量编译的真正优势：大多数链接步骤只是对象文件的快速串联。如果您只更改一个源文件并重建，那么您几乎只需要快速链接步骤，因此您可以在实际开发过程中始终看到这种加速。

如上所述，构建时间更改将因项目而异。在比BananaBread更小的项目中，链接时间加速可能更小，而在更大的项目中，它可能更大。另一个因素是优化：如上所述，测试链接到`-O0`，但对于发布版本，您将需要`-O3`可能，在这种情况下，Emscripten将在最终的WebAssembly上调用Binaryen优化器，运行[元大](https://hacks.mozilla.org/2018/01/shrinking-webassembly-and-javascript-code-sizes-in-emscripten/)，以及其他对代码大小和速度有用的东西。当然，这需要额外的时间，对于发布版本来说，这是值得的 - 在BananaBread上，它将WebAssembly从2.65 MB缩小到1.84 MB，这一改进超过了**30%**— 但对于快速增量构建，您可以使用`-O0`.

## 已知问题

虽然LLVM WebAssembly后端通常在代码大小和速度上都获胜，但我们已经看到了一些例外：

*   [法斯塔](https://github.com/emscripten-core/emscripten/blob/incoming/tests/fasta.cpp)倒退没有[非捕获浮点数到整数转换](https://github.com/WebAssembly/nontrapping-float-to-int-conversions)，这是 WebAssembly MVP 中没有的新 WebAssembly 功能。根本问题是，在 MVP 中，如果浮点数到 int 转换超出有效整数的范围，则会捕获它。理由是，无论如何，这在C中都是未定义的行为，并且对于VM来说很容易实现。但是，事实证明，这与LLVM编译浮点型到int转换的方式不匹配，结果是需要额外的保护，增加了代码大小和开销。较新的非陷印操作可以避免这种情况，但可能尚未出现在所有浏览器中。您可以通过编译源文件来使用它们`-mnontrapping-fptoint`.
*   LLVM WebAssembly后端不仅是与fastcomp不同的后端，而且还使用更新的LLVM。较新的LLVM可能会做出不同的内联决策，这些决策（就像在没有配置文件引导优化的情况下的所有内联决策一样）是启发式驱动的，最终可能会帮助或伤害。我们之前提到的一个具体例子是在LZMA基准测试中，较新的LLVM最终以一种最终只会造成伤害的方式将函数插入5次。如果您在自己的项目中遇到这种情况，则可以有选择地使用`-Os`要专注于代码大小，请使用`__attribute__((noinline))`等。

可能还有更多我们不知道的问题需要优化 - 如果您发现任何东西，请告诉我们！

## 其他更改

有少量的Emscripten功能与fastcomp和/或asm.js相关联，这意味着它们无法与WebAssembly后端一起开箱即用，因此我们一直在研究替代方案。

### JavaScript 输出

在某些情况下，非WebAssembly输出的选项仍然很重要 - 尽管所有主流浏览器都支持WebAssembly一段时间，但仍然有很长一段时间的旧机器，旧手机等没有WebAssembly支持。此外，随着WebAssembly添加新功能，这个问题的某种形式将保持相关性。编译到JS是一种保证你可以接触到每个人的方式，即使构建不像WebAssembly那么小或那么快。对于 fastcomp，我们直接使用 asm.js 输出，但是使用 WebAssembly 后端显然需要其他东西。我们正在使用Binaryen的[`wasm2js`](https://github.com/WebAssembly/binaryen#wasm2js)出于这个目的，顾名思义，它将WebAssembly编译为JS。

这可能值得一篇完整的博客文章，但简而言之，这里的一个关键设计决策是，不再支持asm.js了。asm.js可以比一般的JS运行得快得多，但事实证明，几乎所有支持asm.js AOT优化的浏览器也都支持WebAssembly（事实上，Chrome通过将其内部转换为WebAssembly来优化asm.js！因此，当我们谈论JS回退选项时，它最好不要使用asm.js;事实上，它更简单，允许我们在WebAssembly中支持更多功能，并且还导致JS明显更小！因此`wasm2js`不以 asm 为目标.js。

但是，该设计的副作用是，如果您测试asm.js与使用WebAssembly后端的JS构建相比，从fastcomp构建，那么asm.js可能会快得多 - 如果您在使用asm.js AOT优化的现代浏览器中进行测试。对于您自己的浏览器来说，情况可能就是这样，但实际上不需要非WebAssembly选项的浏览器却不是这样！为了进行适当的比较，您应该使用没有asm.js优化或禁用它们的浏览器。如果`wasm2js`输出仍然较慢，请告诉我们！

`wasm2js`缺少一些较少使用的功能，如动态链接和 pthreads，但大多数代码应该已经工作了，并且已经过仔细的模糊处理。要测试 JS 输出，只需使用`-s WASM=0`以禁用 WebAssembly。`emcc`然后运行`wasm2js`对于您来说，如果这是一个优化的构建，它也会运行各种有用的优化。

### 您可能会注意到的其他事项

*   这[异步](https://github.com/emscripten-core/emscripten/wiki/Asyncify)和[埃莫普特](https://github.com/emscripten-core/emscripten/wiki/Emterpreter)选项仅适用于 fastcomp。替代品[是](https://github.com/WebAssembly/binaryen/pull/2172) [存在](https://github.com/WebAssembly/binaryen/pull/2173) [工作](https://github.com/emscripten-core/emscripten/pull/8808) [上](https://github.com/emscripten-core/emscripten/issues/8561).我们预计这最终将成为对先前选项的改进。
*   必须重建预建库：如果您有一些`library.bc`这是用fastcomp构建的，那么你需要使用更新的Emscripten从源代码重建它。当 fastcomp 将 LLVM 升级到更改了位码格式的新版本时，情况一直如此，现在更改（更改为 WebAssembly 对象文件而不是位码）具有相同的效果。

## 结论

我们现在的主要目标是修复与此更改相关的任何错误。请测试和归档问题！

在事情稳定下来之后，我们将默认的编译器后端切换到上游的WebAssembly后端。如前所述，Fastcomp仍将是一种选择。

我们希望最终完全删除 fastcomp。这样做可以消除重大的维护负担，使我们能够更多地关注WebAssembly后端中的新功能，加速Emscripten的一般改进以及其他好东西。请让我们知道测试如何在您的代码库上进行，以便我们可以开始计划快速comp删除的时间表。

### 谢谢

感谢所有参与LLVM WebAssembly后端开发的人员，`wasm-ld`，Binaryen，Emscripten以及本文中提到的其他内容！这些令人敬畏的人的部分列表是：aardappel，aheejin，alexcrichton，dschuff，jfbastien，jgravelle，nwilson，sbc100，sunfish，tlively，yurydelendik。
