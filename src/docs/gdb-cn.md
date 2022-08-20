***

## 标题： “使用 GDB 调试内置”&#xA;描述： “从 V8 v6.9 开始，可以在 GDB 中创建断点来调试 CSA / ASM / 扭矩内置。

从 V8 v6.9 开始，可以在 GDB（可能还有其他调试器）中创建断点来调试 CSA / ASM / 扭矩内置。

    (gdb) tb i::Isolate::Init
    Temporary breakpoint 1 at 0x7ffff706742b: i::Isolate::Init. (2 locations)
    (gdb) r
    Thread 1 "d8" hit Temporary breakpoint 1, 0x00007ffff7c55bc0 in Isolate::Init
    (gdb) br Builtins_RegExpPrototypeExec
    Breakpoint 2 at 0x7ffff7ac8784
    (gdb) c
    Thread 1 "d8" hit Breakpoint 2, 0x00007ffff7ac8784 in Builtins_RegExpPrototypeExec ()

请注意，使用临时断点（快捷方式）效果很好`tb`在 GDB 中），而不是常规断点 （`br`），因为您只需要在进程启动时使用它。

内置功能在堆栈跟踪中也可见：

    (gdb) bt
    #0  0x00007ffff7ac8784 in Builtins_RegExpPrototypeExec ()
    #1  0x00007ffff78f5066 in Builtins_ArgumentsAdaptorTrampoline ()
    #2  0x000039751d2825b1 in ?? ()
    #3  0x000037ef23a0fa59 in ?? ()
    #4  0x0000000000000000 in ?? ()

警告：

*   仅适用于嵌入式内置组件。
*   断点只能在内置的开头设置。
*   中的初始断点`Isolate::Init`在设置内置断点之前需要，因为 GDB 修改了二进制文件，并且我们在启动时验证了二进制文件中内置部分的哈希值。否则，V8 会抱怨哈希不匹配：

        # Fatal error in ../../src/isolate.cc, line 117
        # Check failed: d.Hash() == d.CreateHash() (11095509419988753467 vs. 3539781814546519144).
