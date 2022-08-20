***

## 标题： 'GDB JIT编译接口集成'&#xA;描述： 'GDB JIT编译接口集成允许V8为GDB提供从V8运行时发出的本机代码的符号和调试信息。

GDB JIT 编译接口集成允许 V8 为 GDB 提供从 V8 运行时发出的本机代码的符号和调试信息。

当 GDB JIT 编译接口被禁用时，GDB 中的典型回溯包含标记为`??`.这些帧对应于动态生成的代码：

    #8  0x08281674 in v8::internal::Runtime_SetProperty (args=...) at src/runtime.cc:3758
    #9  0xf5cae28e in ?? ()
    #10 0xf5cc3a0a in ?? ()
    #11 0xf5cc38f4 in ?? ()
    #12 0xf5cbef19 in ?? ()
    #13 0xf5cb09a2 in ?? ()
    #14 0x0809e0a5 in v8::internal::Invoke (construct=false, func=..., receiver=..., argc=0, args=0x0,
        has_pending_exception=0xffffd46f) at src/execution.cc:97

但是，启用 GDB JIT 编译接口允许 GDB 生成更翔实的堆栈跟踪：

    #6  0x082857fc in v8::internal::Runtime_SetProperty (args=...) at src/runtime.cc:3758
    #7  0xf5cae28e in ?? ()
    #8  0xf5cc3a0a in loop () at test.js:6
    #9  0xf5cc38f4 in test.js () at test.js:13
    #10 0xf5cbef19 in ?? ()
    #11 0xf5cb09a2 in ?? ()
    #12 0x0809e1f9 in v8::internal::Invoke (construct=false, func=..., receiver=..., argc=0, args=0x0,
        has_pending_exception=0xffffd44f) at src/execution.cc:97

GDB 仍然未知的帧对应于没有源信息的本机代码。看[已知限制](#known-limitations)了解更多详情。

GDB JIT 编译接口在 GDB 文档中指定：<https://sourceware.org/gdb/current/onlinedocs/gdb/JIT-Interface.html>

## 先决条件

*   V8 v3.0.9 或更高版本
*   GDB 7.0 或更高版本
*   Linux OS
*   采用英特尔兼容架构（ia32 或 x64）的 CPU

## 启用 GDB JIT 编译接口

默认情况下，GDB JIT 编译接口当前已从编译中排除，并在运行时禁用。要启用它，请执行以下操作：

1.  构建 V8 库`ENABLE_GDB_JIT_INTERFACE`定义。如果你正在使用scons来构建V8，请运行它`gdbjit=on`.
2.  通过`--gdbjit`标记在启动 V8 时。

要检查您是否已正确启用 GDB JIT 集成，请尝试在 上设置断点`__jit_debug_register_code`.调用此函数以通知 GDB 有关新代码对象的信息。

## 已知限制

*   JIT接口的GDB端目前（从GDB 7.2开始）不能非常有效地处理代码对象的注册。每次下一次注册需要更多时间：对于500个注册对象，每次下一次注册需要超过50ms，有1000个注册代码对象 - 超过300 ms。这个问题是[向 GDB 开发者报告](https://sourceware.org/ml/gdb/2011-01/msg00002.html)但目前没有可用的解决方案。为了减轻GDB的压力，目前GDB的实现JIT集成以两种模式运行：*违约*和*满*（启用者`--gdbjit-full`标志）。在*违约*模式 V8 仅通知 GDB 有关附加了源信息的代码对象（通常包括所有用户脚本）。在*满*- 关于所有生成的代码对象（存根，IC，蹦床）。

*   在 x64 上 GDB 无法正确展开堆栈`.eh_frame`部分 （[第 1053 期](https://bugs.chromium.org/p/v8/issues/detail?id=1053))

*   GDB 未收到有关从快照反序列化代码的通知 （[第 1054 期](https://bugs.chromium.org/p/v8/issues/detail?id=1054))

*   仅支持英特尔兼容 CPU 上的 Linux 操作系统。对于不同的操作系统，要么生成不同的ELF标头，要么使用完全不同的对象格式。

*   启用 GDB JIT 接口将禁用压缩 GC。这样做是为了减轻GDB的压力，因为取消注册和注册每个移动的代码对象将产生相当大的开销。

*   GDB JIT 集成仅提供*近似*源信息。它不提供有关局部变量，函数参数，堆栈布局等的任何信息。它不支持单步执行 JavaScript 代码或在给定行上设置断点。但是，可以通过函数的名称在函数上设置断点。
