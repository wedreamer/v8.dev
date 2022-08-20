***

## 标题： '文档'&#xA;描述： “V8 项目的文档。

V8是Google的开源高性能JavaScript和WebAssembly引擎，用C++编写。它用于Chrome和Node.js等。

本文档面向希望在其应用程序中使用 V8 的C++开发人员，以及对 V8 的设计和性能感兴趣的任何人。本文档向您介绍 V8，而其余文档将向您展示如何在代码中使用 V8，并描述其一些设计细节，并提供一组用于衡量 V8 性能的 JavaScript 基准测试。

## 关于V8

V8 实现<a href="https://tc39.es/ecma262/">ECMAScript</a>和<a href="https://webassembly.github.io/spec/core/">WebAssembly</a>，并在 Windows 7 或更高版本、macOS 10.12+ 以及使用 x64、IA-32 或 ARM 处理器的 Linux 系统上运行。其他系统（IBM i、AIX）和处理器（MIPS、ppcle64、s390x）由外部维护，请参见[港口](/docs/ports).V8 可以独立运行，也可以嵌入到任何C++应用程序中。

V8 编译并执行 JavaScript 源代码，处理对象的内存分配，并对不再需要的对象进行垃圾回收。V8 的世代相传的精确垃圾回收器是 V8 性能的关键之一。

JavaScript 通常用于浏览器中的客户端脚本，例如用于操作文档对象模型 （DOM） 对象。然而，DOM通常不是由JavaScript引擎提供的，而是由浏览器提供的。V8 也是如此 — Google Chrome 提供了 DOM。但是 V8 确实提供了 ECMA 标准中指定的所有数据类型、运算符、对象和函数。

V8 使任何C++应用程序都可以向 JavaScript 代码公开自己的对象和函数。由您决定要向 JavaScript 公开的对象和函数。

## 文档概述

*   [从源代码构建 V8](/docs/build)
    *   [查看 V8 源代码](/docs/source-code)
    *   [使用 GN 进行构建](/docs/build-gn)
    *   [针对 ARM/Android 的交叉编译和调试](/docs/cross-compile-arm)
    *   [针对 iOS 的交叉编译](/docs/cross-compile-ios)
    *   [图形用户界面和集成开发环境设置](/docs/ide-setup)
    *   [在 Arm64 上编译](/docs/compile-arm64)
*   [贡献](/docs/contribute)
    *   [尊重代码](/docs/respectful-code)
    *   [V8 的公共 API 及其稳定性](/docs/api)
    *   [成为 V8 提交者](/docs/become-committer)
    *   [提交者的责任](/docs/committer-responsibility)
    *   [眨眼网络测试（又名布局测试）](/docs/blink-layout-tests)
    *   [评估代码覆盖率](/docs/evaluate-code-coverage)
    *   [发布流程](/docs/release-process)
    *   [设计评审指南](/docs/design-review-guidelines)
    *   [实现和发布 JavaScript/WebAssembly 语言功能](/docs/feature-launch-process)
    *   [WebAssembly 功能的暂存和发布清单](/docs/wasm-shipping-checklist)
    *   [片状二分法](/docs/flake-bisect)
    *   [港口处理](/docs/ports)
    *   [官方支持](/docs/official-support)
    *   [合并和修补](/docs/merge-patch)
    *   [节点.js集成构建](/docs/node-integration)
    *   [报告安全漏洞](/docs/security-bugs)
    *   [在本地运行基准测试](/docs/benchmarks)
    *   [测试](/docs/test)
    *   [分类问题](/docs/triage-issues)
*   调试
    *   [使用模拟器进行 Arm 调试](/docs/debug-arm)
    *   [针对 ARM/Android 的交叉编译和调试](/docs/cross-compile-arm)
    *   [使用 GDB 调试内置功能](/docs/gdb)
    *   [通过 V8 检查器协议进行调试](/docs/inspector)
    *   [GDB JIT 编译接口集成](/docs/gdb-jit)
    *   [调查内存泄漏](/docs/memory-leaks)
    *   [堆栈跟踪 API](/docs/stack-trace-api)
    *   [使用 D8](/docs/d8)
    *   [V8 工具](https://v8.dev/tools)
*   嵌入 V8
    *   [嵌入 V8 指南](/docs/embed)
    *   [版本号](/docs/version-numbers)
    *   [内置函数](/docs/builtin-functions)
    *   [i18n 支持](/docs/i18n)
    *   [不受信任的代码缓解措施](/docs/untrusted-code-mitigations)
*   引擎盖下
    *   [点火](/docs/ignition)
    *   [涡轮风扇](/docs/turbofan)
    *   [扭矩用户手册](/docs/torque)
    *   [写入扭矩内置](/docs/torque-builtins)
    *   [编写 CSA 内置](/docs/csa-builtins)
    *   [添加新的 WebAssembly 操作码](/docs/webassembly-opcode)
    *   [地图，又名“隐藏的类”](/docs/hidden-classes)
    *   [松弛跟踪 - 它是什么？](/blog/slack-tracking)
    *   [WebAssembly 编译管道](/docs/wasm-compilation-pipeline)
*   编写可优化的 JavaScript
    *   [使用 V8 的基于样本的分析器](/docs/profile)
    *   [使用 V8 分析铬](/docs/profile-chromium)
    *   [使用 Linux`perf`与 V8](/docs/linux-perf)
    *   [跟踪 V8](/docs/trace)
    *   [使用运行时调用统计信息](/docs/rcs)
