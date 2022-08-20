***

## 标题： 'V8的公共API'&#xA;描述：“本文档讨论了 V8 公共 API 的稳定性，以及开发人员如何对其进行更改。

本文档讨论 V8 公共 API 的稳定性，以及开发人员如何对其进行更改。

## 原料药稳定性

如果铬金丝雀中的V8被证明是崩溃的，它将回滚到以前的金丝雀的V8版本。因此，保持 V8 的 API 从一个金丝雀版本到下一个版本的兼容性非常重要。

我们不断运行一个[机器人](https://ci.chromium.org/p/v8/builders/luci.v8.ci/Linux%20V8%20API%20Stability)这表示 API 稳定性违规。它编译了Chromium的HEAD和V8的[当前金丝雀版本](https://chromium.googlesource.com/v8/v8/+/refs/heads/canary).

此机器人的故障目前仅 FYI，无需执行任何操作。责备列表可用于在回滚时轻松识别从属 CL。

如果打破此机器人，请提醒下次增加 V8 更改和从属 Chromium 更改之间的窗口。

## 如何更改 V8 的公共 API

V8被许多不同的嵌入器使用：Chrome，Node.js，gjstest等。当更改 V8 的公共 API（基本上是`include/`目录），我们需要确保嵌入器可以顺利更新到新的V8版本。特别是，我们不能假设嵌入器更新到新的V8版本，并在一次原子更改中将其代码调整为新的API。

嵌入器应该能够根据新的 API 调整其代码，同时仍然使用以前版本的 V8。以下所有说明均遵循此规则。

*   添加新的类型、常量和函数是安全的，但有一点需要注意：不要向现有类添加新的纯虚函数。新的虚函数应具有默认实现。

*   如果新参数具有默认值，则向函数添加新参数是安全的。

*   删除或重命名类型、常量和函数是不安全的。使用[`V8_DEPRECATED`](https://cs.chromium.org/chromium/src/v8/include/v8config.h?l=395\&rcl=0425b20ad9a8ba38c2e0dd16e8814abb722bfdde)和[`V8_DEPRECATE_SOON`](https://cs.chromium.org/chromium/src/v8/include/v8config.h?l=403\&rcl=0425b20ad9a8ba38c2e0dd16e8814abb722bfdde)宏，当嵌入程序调用已弃用的方法时，这会导致编译时警告。例如，假设我们要重命名函数`foo`到功能`bar`.然后，我们需要执行以下操作：

    *   添加新函数`bar`靠近现有函数`foo`.
    *   等到 CL 在 Chrome 中滚动。调整要使用的 Chrome`bar`.
    *   注释`foo`跟`V8_DEPRECATED("Use bar instead") void foo();`
    *   在同一 CL 中调整使用`foo`使用`bar`.
    *   在 CL 中编写更改动机和高级更新说明。
    *   等到下一个 V8 分支。
    *   删除功能`foo`.

    `V8_DEPRECATE_SOON`是更柔和的版本`V8_DEPRECATED`.Chrome不会与它一起中断，因此不需要步骤b。`V8_DEPRECATE_SOON`不足以删除该函数。

    您仍然需要注释`V8_DEPRECATED`并等待下一个分支，然后再删除该函数。

    `V8_DEPRECATED`可以使用`v8_deprecation_warnings`GN 标志。
    `V8_DEPRECATE_SOON`可使用`v8_imminent_deprecation_warnings`.

*   更改函数签名是不安全的。使用`V8_DEPRECATED`和`V8_DEPRECATE_SOON`如上所述的宏。

我们维护一个[提及重要 API 更改的文档](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit)对于每个 V8 版本。

还有一个定期更新[无氧原料药文档](https://v8.dev/api).
