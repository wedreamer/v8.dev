***

标题： '使用 WebAssembly SIMD 的快速并行应用程序'
作者： 'Deepti Gandluri （[@dptig](https://twitter.com/dptig)）， 托马斯·利弗利 （[@tlively52](https://twitter.com/tlively52)）， 英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
日期： 2020-01-30
更新日期： 2022-03-21
标签：

*   WebAssembly
    描述： “将矢量操作引入 WebAssembly”
    推文：“1222944308183085058”

***

SIMD 代表*单指令，多数据*.SIMD 指令是一类特殊的指令，它通过同时对多个数据元素执行相同的操作来利用应用程序中的数据并行性。音频/视频编解码器、图像处理器等计算密集型应用程序都是利用 SIMD 指令提高性能的应用程序示例。大多数现代体系结构都支持 SIMD 指令的某些变体。

WebAssembly SIMD 提案定义了可在大多数现代体系结构中使用的 SIMD 操作的可移植、高性能子集。该提案从[单色.js提案](https://github.com/tc39/ecmascript_simd)，而这反过来又最初源自[飞镖单片机](https://www.researchgate.net/publication/261959129\_A_SIMD_programming_model_for_dart_javascriptand_other_dynamically_typed_scripting_languages)规范。SIMD.js提案是 TC39 上提出的一个 API，具有用于执行 SIMD 计算的新类型和功能，但该提案已存档，以便在 WebAssembly 中更透明地支持 SIMD 操作。这[网络组装 SIMD 提案](https://github.com/WebAssembly/simd)作为浏览器使用底层硬件利用数据级并行性的一种方式引入的。

## 网络组装 SIMD 提案

WebAssembly SIMD 提案的高级目标是以一种保证可移植性能的方式将矢量操作引入 WebAssembly 规范。

SIMD 指令集很大，并且因体系结构而异。WebAssembly SIMD 提案中包含的一组操作包括在各种平台上得到良好支持的操作，并且被证明是高性能的。为此，目前的提案仅限于标准化固定宽度 128 位 SIMD 操作。

目前的提案引入了一个新的`v128`值类型，以及对此类型执行操作的许多新操作。用于确定这些操作的条件是：

*   这些操作应该在多个现代体系结构中得到很好的支持。
*   在指令组内的多个相关体系结构中，性能优势应该是积极的。
*   所选的操作集应最大程度地减少性能悬崖（如果有）。

该提案现已发布[最终状态（阶段 4）](https://github.com/WebAssembly/simd/issues/480)，V8 和工具链都有工作实现。

## 启用 SIMD 支持

### 特征检测

首先，请注意，SIMD 是一项新功能，尚未在所有支持 WebAssembly 的浏览器中使用。您可以在上找到哪些浏览器支持新的WebAssembly功能[webassembly.org](https://webassembly.org/roadmap/)网站。

为确保所有用户都可以加载您的应用程序，您需要构建两个不同的版本（一个启用了 SIMD，另一个没有 SIMD），并根据功能检测结果加载相应的版本。要在运行时检测 SIMD，您可以使用[`wasm-feature-detect`](https://github.com/GoogleChromeLabs/wasm-feature-detect)库并加载相应的模块，如下所示：

```js
import { simd } from 'wasm-feature-detect';

(async () => {
  const hasSIMD = await simd();
  const module = await (
    hasSIMD
      ? import('./module-with-simd.js')
      : import('./module-without-simd.js')
  );
  // …now use `module` as you normally would
})();
```

要了解如何使用 SIMD 支持构建代码，请查看该部分[下面](#building-with-simd-support).

### Chrome 中的 SIMD 支持

默认情况下，WebAssembly SIMD 支持可从 Chrome 91 获得。确保使用最新版本的工具链，如下所述，以及最新的 wasm-feature-detect 来检测支持规范最终版本的引擎。如果有什么东西看起来不对劲，请[归档错误](https://crbug.com/v8).

### 在 Firefox 中启用实验性 SIMD 支持

WebAssembly SIMD 在 Firefox 中位于标志后面。目前，它仅在 x86 和 x86-64 体系结构上受支持。要尝试 Firefox 中的 SIMD 支持，请转到`about:config`并启用`javascript.options.wasm_simd`.请注意，此功能仍处于实验阶段，正在开发中。

## 支持 SIMD 进行构建

### 构建 C/C++以目标 SIMD 为目标

WebAssembly 的 SIMD 支持依赖于使用启用了 WebAssembly LLVM 后端的最新版本的 clang。Emscripten也支持WebAssembly SIMD提案。安装并激活`latest`表情符号的分布使用[emsdk](https://emscripten.org/docs/getting_started/downloads.html)以使用 SIMD 功能。

```bash
./emsdk install latest
./emsdk activate latest
```

有几种不同的方法可以在移植应用程序以使用 SIMD 时生成 SIMD 代码。一旦安装了最新的上游脚本版本，使用emscripten编译，然后传递`-msimd128`标记以启用 SIMD。

```bash
emcc -msimd128 -O3 foo.c -o foo.js
```

由于LLVM的自动矢量化优化，已经移植到使用WebAssembly的应用程序可以从SIMD中受益，而无需修改源代码。

这些优化可以将对每次迭代执行算术运算的循环自动转换为等效循环，这些循环使用 SIMD 指令一次对多个输入执行相同的算术运算。默认情况下，LLVM 的自动矢量化器在优化级别处于启用状态`-O2`和`-O3`当`-msimd128`提供标志。

例如，请考虑以下函数，该函数将两个输入数组的元素相乘，并将结果存储在输出数组中。

```cpp
void multiply_arrays(int* out, int* in_a, int* in_b, int size) {
  for (int i = 0; i < size; i++) {
    out[i] = in_a[i] * in_b[i];
  }
}
```

不通过`-msimd128`标记，编译器发出以下 WebAssembly 循环：

```wasm
(loop
  (i32.store
    … get address in `out` …
    (i32.mul
      (i32.load … get address in `in_a` …)
      (i32.load … get address in `in_b` …)
  …
)
```

但是当`-msimd128`使用标志，自动矢量化器将其转换为包含以下循环的代码：

```wasm
(loop
  (v128.store align=4
    … get address in `out` …
    (i32x4.mul
       (v128.load align=4 … get address in `in_a` …)
       (v128.load align=4 … get address in `in_b` …)
    …
  )
)
```

循环体具有相同的结构，但 SIMD 指令用于在循环体内一次加载、乘法和存储四个元素。

要对编译器生成的 SIMD 指令进行更细粒度的控制，请包括[`wasm_simd128.h`头文件](https://github.com/llvm/llvm-project/blob/master/clang/lib/Headers/wasm_simd128.h)，它定义了一组内部函数。内部函数是特殊函数，在调用时，编译器会将其转换为相应的 WebAssembly SIMD 指令，除非它可以进行进一步的优化。

例如，下面是手动重写以使用 SIMD 内部函数之前的相同函数。

```cpp
#include <wasm_simd128.h>

void multiply_arrays(int* out, int* in_a, int* in_b, int size) {
  for (int i = 0; i < size; i += 4) {
    v128_t a = wasm_v128_load(&in_a[i]);
    v128_t b = wasm_v128_load(&in_b[i]);
    v128_t prod = wasm_i32x4_mul(a, b);
    wasm_v128_store(&out[i], prod);
  }
}
```

此手动重写的代码假定输入和输出数组是对齐的，并且不具有别名，并且大小是 4 的倍数。自动矢量化器无法做出这些假设，并且必须生成额外的代码来处理它们不真实的情况，因此手写的 SIMD 代码通常最终比自动矢量化的 SIMD 代码小。

### 交叉编译现有的C /C++项目

许多现有项目在针对其他平台时已经支持 SIMD，特别是[上证](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)和[断续器](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)x86 / x86-64 平台上的说明和[氖](https://en.wikipedia.org/wiki/ARM_architecture#Advanced_SIMD_\(Neon\))ARM 平台上的说明。通常有两种方式实现它们。

第一种是通过处理SIMD操作的汇编文件，并在构建过程中与C / C++链接在一起。程序集语法和指令高度依赖于平台且不可移植，因此，要利用 SIMD，此类项目需要添加 WebAssembly 作为附加支持的目标，并使用[网页组合文本格式](https://webassembly.github.io/spec/core/text/index.html)或描述的内在函数[以上](#building-c-%2F-c%2B%2B-to-target-simd).

另一种常见的方法是直接从C / C++代码中使用SSE / SSE2 / AVX / NEON内部函数，在这里Emscripten可以提供帮助。Emscripten[提供兼容的标头和仿真层](https://emscripten.org/docs/porting/simd.html)对于所有这些指令集，以及一个仿真层，该仿真层在可能的情况下将它们直接编译为Wasm内部函数，或者以其他方式进行标量化代码。

要交叉编译此类项目，请首先通过特定于项目的配置标志启用 SIMD，例如`./configure --enable-simd`以便它通过`-msse`,`-msse2`,`-mavx`或`-mfpu=neon`到编译器并调用相应的内部函数。然后，另外通过`-msimd128`以启用 WebAssembly SIMD，方法是使用`CFLAGS=-msimd128 make …`/`CXXFLAGS="-msimd128 make …`或者在面向 Wasm 时直接修改构建配置。

### 构建 Rust 以针对 SIMD 为目标

在编译 Rust 代码以面向 WebAssembly SIMD 时，您需要启用相同的`simd128`LLVM 功能如上面的 Emscripten 所示。

如果您可以控制`rustc`直接或通过环境变量标记`RUSTFLAGS`通过`-C target-feature=+simd128`:

```bash
rustc … -C target-feature=+simd128 -o out.wasm
```

或

```bash
RUSTFLAGS="-C target-feature=+simd128" cargo build
```

与 Clang / Emscripten 一样，LLVM 的自动矢量化器在以下情况下默认启用，以便优化代码`simd128`功能已启用。

例如，Rust 相当于`multiply_arrays`上面的例子

```rust
pub fn multiply_arrays(out: &mut [i32], in_a: &[i32], in_b: &[i32]) {
  in_a.iter()
    .zip(in_b)
    .zip(out)
    .for_each(|((a, b), dst)| {
        *dst = a * b;
    });
}
```

将为输入的对齐部分生成类似的自动向量化代码。

为了能够手动控制 SIMD 操作，可以使用夜间工具链，启用 Rust 功能`wasm_simd`并从[`std::arch::wasm32`](https://doc.rust-lang.org/stable/core/arch/wasm32/index.html#simd)直接命名空间：

```rust
#![feature(wasm_simd)]

use std::arch::wasm32::*;

pub unsafe fn multiply_arrays(out: &mut [i32], in_a: &[i32], in_b: &[i32]) {
  in_a.chunks(4)
    .zip(in_b.chunks(4))
    .zip(out.chunks_mut(4))
    .for_each(|((a, b), dst)| {
      let a = v128_load(a.as_ptr() as *const v128);
      let b = v128_load(b.as_ptr() as *const v128);
      let prod = i32x4_mul(a, b);
      v128_store(dst.as_mut_ptr() as *mut v128, prod);
    });
}
```

或者，使用辅助板条箱，如[`packed_simd`](https://crates.io/crates/packed_simd\_2)在各种平台上对 SIMD 实现进行抽象。

## 引人注目的用例

WebAssembly SIMD 提案旨在加速高计算应用，如音频/视频编解码器、图像处理应用、加密应用等。目前，WebAssembly SIMD在广泛使用的开源项目中得到了实验性支持，例如[卤化物](https://github.com/halide/Halide/blob/master/README_webassembly.md),[OpenCV.js](https://docs.opencv.org/3.4/d5/d10/tutorial_js_root.html)和[鑫鑫派克](https://github.com/google/XNNPACK).

一些有趣的演示来自[MediaPipe项目](https://github.com/google/mediapipe)由谷歌研究团队提供。

根据他们的描述，MediaPipe是一个用于构建多模式（例如视频，音频，任何时间序列数据）应用的ML管道的框架。他们有一个[网页版](https://developers.googleblog.com/2020/01/mediapipe-on-web.html)太！

在视觉上最吸引人的演示中，很容易观察到 SIMD 性能的差异，它是手部跟踪系统的纯 CPU（非 GPU）构建。[不含单片处理](https://storage.googleapis.com/aim-bucket/users/tmullen/demos\_10\_2019\_cdc/rebuild\_04\_2021/mediapipe_handtracking/gl_graph_demo.html)，在现代笔记本电脑上只能获得大约14-15 FPS（每秒帧数），而[在 Chrome Canary 中启用了 SIMD](https://storage.googleapis.com/aim-bucket/users/tmullen/demos\_10\_2019\_cdc/rebuild\_04\_2021/mediapipe_handtracking_simd/gl_graph_demo.html)您可以在38-40 FPS下获得更流畅的体验。

<figure>
  <video autoplay muted playsinline loop width="600" height="216" src="/_img/simd/hand.mp4"></video>
</figure>

另一组有趣的演示，利用SIMD获得流畅的体验，来自OpenCV - 一个流行的计算机视觉库，也可以编译为WebAssembly。它们可通过以下方式获得[链接](https://bit.ly/opencv-camera-demos)，或者您可以查看以下预先录制的版本：

<figure>
  <video autoplay muted playsinline loop width="256" height="512" src="/_img/simd/credit-card.mp4"></video>
  <figcaption>Card reading</figcaption>
</figure>

<figure>
  <video autoplay muted playsinline loop width="600" height="646" src="/_img/simd/invisibility-cloak.mp4"></video>
  <figcaption>Invisibility cloak</figcaption>
</figure>

<figure>
  <video autoplay muted playsinline loop width="600" height="658" src="/_img/simd/emotion-recognizer.mp4"></video>
  <figcaption>Emoji replacement</figcaption>
</figure>

## 三. 今后的工作

当前的固定宽度 SIMD 提案位于[第 4 阶段](https://github.com/WebAssembly/meetings/blob/master/process/phases.md#3-implementation-phase-community--working-group)，因此它被认为是完整的。

对未来 SIMD 扩展的一些探索已在[轻松的单色单片机](https://github.com/WebAssembly/relaxed-simd)和[柔性载体](https://github.com/WebAssembly/flexible-vectors)在撰写本文时，提案处于第1阶段。
