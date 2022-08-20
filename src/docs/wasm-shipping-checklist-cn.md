***

## 标题：“WebAssembly 功能的暂存和交付清单”&#xA;描述： “本文档提供了有关何时在 V8 中暂存和发布 WebAssembly 功能的工程要求清单。

本文档提供了在 V8 中暂存和发布 WebAssembly 功能的工程要求清单。这些清单仅供参考，可能不适用于所有功能。实际的启动过程在[V8 启动过程](https://v8.dev/docs/feature-launch-process).

# 分期

## 何时暂存 WebAssembly 功能

这[分期](https://docs.google.com/document/d/1ZgyNx7iLtRByBtbYi1GssWGefXXciLeADZBR_FxG-hE)的 WebAssembly 功能定义了其实现阶段的结束。完成以下清单后，实施阶段即告完成：

*   V8 中的实现已完成。这包括：
    *   在涡轮风扇中实施（如果适用）
    *   在 Liftoff 中实现（如果适用）
    *   在解释器中实现（如果适用）
*   V8 中的测试可用
*   规范测试通过运行[`tools/wasm/update-wasm-spec-tests.sh`](https://cs.chromium.org/chromium/src/v8/tools/wasm/update-wasm-spec-tests.sh)
*   所有现有提案规范测试均通过。缺少规范测试是不幸的，但不应阻止暂存。

请注意，标准化过程中功能建议的阶段对于在 V8 中暂存功能并不重要。然而，该提案应基本上是稳定的。

## 如何暂存 WebAssembly 功能

*   在[`src/wasm/wasm-feature-flags.h`](https://cs.chromium.org/chromium/src/v8/src/wasm/wasm-feature-flags.h)，将功能标志从`FOREACH_WASM_EXPERIMENTAL_FEATURE_FLAG`宏列表到`FOREACH_WASM_STAGING_FEATURE_FLAG`宏列表。
*   在[`tools/wasm/update-wasm-spec-tests.sh`](https://cs.chromium.org/chromium/src/v8/tools/wasm/update-wasm-spec-tests.sh)，将建议存储库名称添加到`repos`存储库列表。
*   跑[`tools/wasm/update-wasm-spec-tests.sh`](https://cs.chromium.org/chromium/src/v8/tools/wasm/update-wasm-spec-tests.sh)以创建并上传新提案的规范测试。
*   在[`test/wasm-spec-tests/testcfg.py`](https://cs.chromium.org/chromium/src/v8/test/wasm-spec-tests/testcfg.py)，将建议存储库名称和功能标志添加到`proposal_flags`列表。
*   在[`test/wasm-js/testcfg.py`](https://cs.chromium.org/chromium/src/v8/test/wasm-js/testcfg.py)，将建议存储库名称和功能标志添加到`proposal_flags`列表。

查看[类型反射的暂存](https://crrev.com/c/1771791)作为参考。

# 航运

## WebAssembly 功能何时准备就绪

*   这[V8 启动过程](https://v8.dev/docs/feature-launch-process)很满意。
*   该实现由模糊器（如果适用）覆盖。
*   该功能已经上演了几个星期，以获得模糊覆盖。
*   功能建议是[阶段 4](https://github.com/WebAssembly/proposals).
*   都[规格测试](https://github.com/WebAssembly/spec/tree/master/test)通过。
*   这[Chromium DevTools 核對清單，以針對新的 WebAssembly 功能](https://docs.google.com/document/d/1WbL-fGuLbbNr5-n_nRGo_ILqZFnh5ZjRSUcDTT3yI8s/preview)很满意。

## 如何发布 WebAssembly 功能

*   在[`src/wasm/wasm-feature-flags.h`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/wasm/wasm-feature-flags.h)，将功能标志从`FOREACH_WASM_STAGING_FEATURE_FLAG`宏列表到`FOREACH_WASM_SHIPPED_FEATURE_FLAG`宏列表。
    *   确保在 CL 上添加一个眨眼 CQ 机器人以进行检查[眨眼网络测试](https://v8.dev/docs/blink-layout-tests)因启用该功能而导致的故障（将此行添加到 CL 描述的页脚：`Cq-Include-Trybots: luci.v8.try:v8_linux_blink_rel`).
*   此外，默认情况下，通过更改 中的第三个参数来启用该功能`FOREACH_WASM_SHIPPED_FEATURE_FLAG`自`true`.
*   设置提醒以在两个里程碑后删除功能标志。
