***

## 标题： “使用模拟器进行 Arm 调试”&#xA;描述： “Arm模拟器和调试器在使用V8代码生成时非常有用。

模拟器和调试器在使用 V8 代码生成时非常有用。

*   它很方便，因为它允许您在不访问实际硬件的情况下测试代码生成。
*   不[渡](/docs/cross-compile-arm)或需要本机编译。
*   模拟器完全支持调试生成的代码。

请注意，此模拟器专为 V8 而设计。只有 V8 使用的功能部件才实现，您可能会遇到未实现的功能或指令。在这种情况下，请随时实现它们并提交代码！

*   [编译](#compiling)
*   [启动调试器](#start_debug)
*   [调试命令](#debug_commands)
    *   [打印对象](#po)
    *   [跟踪](#trace)
    *   [破](#break)
*   [额外的断点功能](#extra)
    *   [32 位：`stop()`](#arm32\_stop)
    *   [64 位：`Debug()`](#arm64\_debug)

## 使用模拟器为 Arm 编译 { #compiling }

默认情况下，在 x86 主机上，编译为 Arm[通用 汽车](/docs/build-gn#gm)将为您提供模拟器构建：

```bash
gm arm64.debug # For a 64-bit build or...
gm arm.debug   # ... for a 32-bit build.
```

您还可以构建`optdebug`配置为`debug`可能有点慢，特别是如果你想运行V8测试套件。

## 启动调试器 { #start_debug }

您可以在以下位置后立即从命令行启动调试器`n`指示：

```bash
out/arm64.debug/d8 --stop_sim_at <n> # Or out/arm.debug/d8 for a 32-bit build.
```

或者，您可以在生成的代码中生成断点指令：

在本机，断点指令会导致程序停止，并带有`SIGTRAP`信号，允许您使用 gdb 调试问题。但是，如果使用模拟器运行，则生成的代码中的断点指令将改为将您放入模拟器调试器中。

您可以使用`DebugBreak()`从[力矩](/docs/torque-builtins)，从[CodeStubAssembler](/docs/csa-builtins)，作为[涡轮风扇](/docs/turbofan)通过，或直接使用汇编程序。

这里我们专注于调试低级本机代码，因此让我们看一下汇编器方法：

```cpp
TurboAssembler::DebugBreak();
```

假设我们有一个名为`add`编译[涡轮风扇](/docs/turbofan)我们想在一开始就打破。给定一个`test.js`例：

{ #test.js }

```js
// Our optimized function.
function add(a, b) {
  return a + b;
}

// Typical cheat code enabled by --allow-natives-syntax.
%PrepareFunctionForOptimization(add);

// Give the optimizing compiler type feedback so it'll speculate `a` and `b` are
// numbers.
add(1, 3);

// And force it to optimize.
%OptimizeFunctionOnNextCall(add);
add(5, 7);
```

要做到这一点，我们可以连接到TurboFan的[代码生成器](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/backend/code-generator.cc?q=CodeGenerator::AssembleCode)并访问汇编程序以插入我们的断点：

```cpp
void CodeGenerator::AssembleCode() {
  // ...

  // Check if we're optimizing, then look-up the name of the current function and
  // insert a breakpoint.
  if (info->IsOptimizing()) {
    AllowHandleDereference allow_handle_dereference;
    if (info->shared_info()->PassesFilter("add")) {
      tasm()->DebugBreak();
    }
  }

  // ...
}
```

让我们运行它：

```simulator
$ d8 \
    # Enable '%' cheat code JS functions.
    --allow-natives-syntax \
    # Disassemble our function.
    --print-opt-code --print-opt-code-filter="add" --code-comments \
    # Disable spectre mitigations for readability.
    --no-untrusted-code-mitigations \
    test.js
--- Raw source ---
(a, b) {
  return a + b;
}


--- Optimized code ---
optimization_id = 0
source_position = 12
kind = OPTIMIZED_FUNCTION
name = add
stack_slots = 6
compiler = turbofan
address = 0x7f0900082ba1

Instructions (size = 504)
0x7f0900082be0     0  d45bd600       constant pool begin (num_const = 6)
0x7f0900082be4     4  00000000       constant
0x7f0900082be8     8  00000001       constant
0x7f0900082bec     c  75626544       constant
0x7f0900082bf0    10  65724267       constant
0x7f0900082bf4    14  00006b61       constant
0x7f0900082bf8    18  d45bd7e0       constant
                  -- Prologue: check code start register --
0x7f0900082bfc    1c  10ffff30       adr x16, #-0x1c (addr 0x7f0900082be0)
0x7f0900082c00    20  eb02021f       cmp x16, x2
0x7f0900082c04    24  54000080       b.eq #+0x10 (addr 0x7f0900082c14)
                  Abort message:
                  Wrong value in code start register passed
0x7f0900082c08    28  d2800d01       movz x1, #0x68
                  -- Inlined Trampoline to Abort --
0x7f0900082c0c    2c  58000d70       ldr x16, pc+428 (addr 0x00007f0900082db8)    ;; off heap target
0x7f0900082c10    30  d63f0200       blr x16
                  -- Prologue: check for deoptimization --
                  [ DecompressTaggedPointer
0x7f0900082c14    34  b85d0050       ldur w16, [x2, #-48]
0x7f0900082c18    38  8b100350       add x16, x26, x16
                  ]
0x7f0900082c1c    3c  b8407210       ldur w16, [x16, #7]
0x7f0900082c20    40  36000070       tbz w16, #0, #+0xc (addr 0x7f0900082c2c)
                  -- Inlined Trampoline to CompileLazyDeoptimizedCode --
0x7f0900082c24    44  58000c31       ldr x17, pc+388 (addr 0x00007f0900082da8)    ;; off heap target
0x7f0900082c28    48  d61f0220       br x17
                  -- B0 start (construct frame) --
(...)

--- End code ---
# Debugger hit 0: DebugBreak
0x00007f0900082bfc 10ffff30            adr x16, #-0x1c (addr 0x7f0900082be0)
sim>
```

我们可以看到，在优化功能开始时，我们已经停止了，模拟器给了我们一个提示！

请注意，这只是一个示例，V8 变化很快，因此细节可能会有所不同。但是，您应该能够在任何有汇编程序的地方执行此操作。

## 调试命令 { #debug_commands }

### 常用命令

进入`help`以获取有关可用命令的详细信息。这些包括通常的类似 gdb 的命令，例如`stepi`,`cont`,`disasm`等。如果模拟器在 gdb 下运行，则`gdb`调试器命令将控制权交给 gdb。然后，您可以使用`cont`从 gdb 返回到调试器。

### 特定于体系结构的命令

每个目标体系结构都实现自己的模拟器和调试器，因此体验和详细信息会有所不同。

*   [打印对象](#po)
*   [跟踪](#trace)
*   [破](#break)

#### `printobject $register`（别名`po`） { #po }

描述保存在寄存器中的 JS 对象。

例如，假设这次我们正在运行[我们的例子](#test.js)在 32 位 Arm 模拟器构建上。我们可以检查寄存器中传递的传入参数：

```simulator
$ ./out/arm.debug/d8 --allow-natives-syntax test.js
Simulator hit stop, breaking at the next instruction:
  0x26842e24  e24fc00c       sub ip, pc, #12
sim> print r1
r1: 0x4b60ffb1 1264648113
# The current function object is passed with r1.
sim> printobject r1
r1:
0x4b60ffb1: [Function] in OldSpace
 - map: 0x485801f9 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x4b6010f1 <JSFunction (sfi = 0x42404e99)>
 - elements: 0x5b700661 <FixedArray[0]> [HOLEY_ELEMENTS]
 - function prototype:
 - initial_map:
 - shared_info: 0x4b60fe9d <SharedFunctionInfo add>
 - name: 0x5b701c5d <String[#3]: add>
 - formal_parameter_count: 2
 - kind: NormalFunction
 - context: 0x4b600c65 <NativeContext[261]>
 - code: 0x26842de1 <Code OPTIMIZED_FUNCTION>
 - source code: (a, b) {
  return a + b;
}
(...)

# Now print the current JS context passed in r7.
sim> printobject r7
r7:
0x449c0c65: [NativeContext] in OldSpace
 - map: 0x561000b9 <Map>
 - length: 261
 - scope_info: 0x34081341 <ScopeInfo SCRIPT_SCOPE [5]>
 - previous: 0
 - native_context: 0x449c0c65 <NativeContext[261]>
           0: 0x34081341 <ScopeInfo SCRIPT_SCOPE [5]>
           1: 0
           2: 0x449cdaf5 <JSObject>
           3: 0x58480c25 <JSGlobal Object>
           4: 0x58485499 <Other heap object (EMBEDDER_DATA_ARRAY_TYPE)>
           5: 0x561018a1 <Map(HOLEY_ELEMENTS)>
           6: 0x3408027d <undefined>
           7: 0x449c75c1 <JSFunction ArrayBuffer (sfi = 0x4be8ade1)>
           8: 0x561010f9 <Map(HOLEY_ELEMENTS)>
           9: 0x449c967d <JSFunction arrayBufferConstructor_DoNotInitialize (sfi = 0x4be8c3ed)>
          10: 0x449c8dbd <JSFunction Array (sfi = 0x4be8be59)>
(...)
```

#### `trace`（别名`t`） { #trace }

启用或禁用跟踪执行的指令。

启用后，模拟器将在执行拆卸指令时打印这些指令。如果运行的是 64 位 Arm 版本，模拟器还能够跟踪对寄存器值的更改。

您也可以从命令行启用此功能，并使用`--trace-sim`标记以从头开始启用跟踪。

与[例](#test.js):

```simulator
$ out/arm64.debug/d8 --allow-natives-syntax \
    # --debug-sim is required on 64-bit Arm to enable disassembly
    # when tracing.
    --debug-sim test.js
# Debugger hit 0: DebugBreak
0x00007f1e00082bfc  10ffff30            adr x16, #-0x1c (addr 0x7f1e00082be0)
sim> trace
0x00007f1e00082bfc  10ffff30            adr x16, #-0x1c (addr 0x7f1e00082be0)
Enabling disassembly, registers and memory write tracing

# Break on the return address stored in the lr register.
sim> break lr
Set a breakpoint at 0x7f1f880abd28
0x00007f1e00082bfc  10ffff30            adr x16, #-0x1c (addr 0x7f1e00082be0)

# Continuing will trace the function's execution until we return, allowing
# us to make sense of what is happening.
sim> continue
#    x0: 0x00007f1e00082ba1
#    x1: 0x00007f1e08250125
#    x2: 0x00007f1e00082be0
(...)

# We first load the 'a' and 'b' arguments from the stack and check if they
# are tagged numbers. This is indicated by the least significant bit being 0.
0x00007f1e00082c90  f9401fe2            ldr x2, [sp, #56]
#    x2: 0x000000000000000a <- 0x00007f1f821f0278
0x00007f1e00082c94  7200005f            tst w2, #0x1
# NZCV: N:0 Z:1 C:0 V:0
0x00007f1e00082c98  54000ac1            b.ne #+0x158 (addr 0x7f1e00082df0)
0x00007f1e00082c9c  f9401be3            ldr x3, [sp, #48]
#    x3: 0x000000000000000e <- 0x00007f1f821f0270
0x00007f1e00082ca0  7200007f            tst w3, #0x1
# NZCV: N:0 Z:1 C:0 V:0
0x00007f1e00082ca4  54000a81            b.ne #+0x150 (addr 0x7f1e00082df4)

# Then we untag and add 'a' and 'b' together.
0x00007f1e00082ca8  13017c44            asr w4, w2, #1
#    x4: 0x0000000000000005
0x00007f1e00082cac  2b830484            adds w4, w4, w3, asr #1
# NZCV: N:0 Z:0 C:0 V:0
#    x4: 0x000000000000000c
# That's 5 + 7 == 12, all good!

# Then we check for overflows and tag the result again.
0x00007f1e00082cb0  54000a46            b.vs #+0x148 (addr 0x7f1e00082df8)
0x00007f1e00082cb4  2b040082            adds w2, w4, w4
# NZCV: N:0 Z:0 C:0 V:0
#    x2: 0x0000000000000018
0x00007f1e00082cb8  54000466            b.vs #+0x8c (addr 0x7f1e00082d44)


# And finally we place the result in x0.
0x00007f1e00082cbc  aa0203e0            mov x0, x2
#    x0: 0x0000000000000018
(...)

0x00007f1e00082cec  d65f03c0            ret
Hit and disabled a breakpoint at 0x7f1f880abd28.
0x00007f1f880abd28  f85e83b4            ldur x20, [fp, #-24]
sim>
```

#### `break $address`{ #break }

在指定地址处插入断点。

请注意，在 32 位 Arm 上，只能有一个断点，需要禁用代码页上的写保护才能插入断点。64 位 Arm 模拟器没有此类限制。

与我们的[例](#test.js)再：

```simulator
$ out/arm.debug/d8 --allow-natives-syntax \
    # This is useful to know which address to break to.
    --print-opt-code --print-opt-code-filter="add" \
    test.js
(...)

Simulator hit stop, breaking at the next instruction:
  0x488c2e20  e24fc00c       sub ip, pc, #12

# Break on a known interesting address, where we start
# loading 'a' and 'b'.
sim> break 0x488c2e9c
sim> continue
  0x488c2e9c  e59b200c       ldr r2, [fp, #+12]

# We can look-ahead with 'disasm'.
sim> disasm 10
  0x488c2e9c  e59b200c       ldr r2, [fp, #+12]
  0x488c2ea0  e3120001       tst r2, #1
  0x488c2ea4  1a000037       bne +228 -> 0x488c2f88
  0x488c2ea8  e59b3008       ldr r3, [fp, #+8]
  0x488c2eac  e3130001       tst r3, #1
  0x488c2eb0  1a000037       bne +228 -> 0x488c2f94
  0x488c2eb4  e1a040c2       mov r4, r2, asr #1
  0x488c2eb8  e09440c3       adds r4, r4, r3, asr #1
  0x488c2ebc  6a000037       bvs +228 -> 0x488c2fa0
  0x488c2ec0  e0942004       adds r2, r4, r4

# And try and break on the result of the first `adds` instructions.
sim> break 0x488c2ebc
setting breakpoint failed

# Ah, we need to delete the breakpoint first.
sim> del
sim> break 0x488c2ebc
sim> cont
  0x488c2ebc  6a000037       bvs +228 -> 0x488c2fa0

sim> print r4
r4: 0x0000000c 12
# That's 5 + 7 == 12, all good!
```

### 生成的断点指令，其中包含一些附加功能 { #extra }

而不是`TurboAssembler::DebugBreak()`，您可以使用较低级别的指令，该指令除了具有其他功能外，还具有相同的效果。

*   [32 位：`stop()`](#arm32\_stop)
*   [64 位：`Debug()`](#arm64\_debug)

#### `stop()`（32 位臂）{ #arm32\_stop }

```cpp
Assembler::stop(Condition cond = al, int32_t code = kDefaultStopCode);
```

第一个参数是条件，第二个参数是停止代码。如果指定了代码，并且小于256，则称为“监视”停止，并且可以禁用/启用;计数器还会跟踪模拟器命中此代码的次数。

想象一下，我们正在开发这个V8 C++代码：

```cpp
__ stop(al, 123);
__ mov(r0, r0);
__ mov(r0, r0);
__ mov(r0, r0);
__ mov(r0, r0);
__ mov(r0, r0);
__ stop(al, 0x1);
__ mov(r1, r1);
__ mov(r1, r1);
__ mov(r1, r1);
__ mov(r1, r1);
__ mov(r1, r1);
```

下面是一个示例调试会话：

我们到达了第一站。

```simulator
Simulator hit stop 123, breaking at the next instruction:
  0xb53559e8  e1a00000       mov r0, r0
```

我们可以看到以下停止使用`disasm`.

```simulator
sim> disasm
  0xb53559e8  e1a00000       mov r0, r0
  0xb53559ec  e1a00000       mov r0, r0
  0xb53559f0  e1a00000       mov r0, r0
  0xb53559f4  e1a00000       mov r0, r0
  0xb53559f8  e1a00000       mov r0, r0
  0xb53559fc  ef800001       stop 1 - 0x1
  0xb5355a00  e1a00000       mov r1, r1
  0xb5355a04  e1a00000       mov r1, r1
  0xb5355a08  e1a00000       mov r1, r1
```

可以打印所有（被监视的）至少被击中一次的停靠点的信息。

```simulator
sim> stop info all
Stop information:
stop 123 - 0x7b:      Enabled,      counter = 1
sim> cont
Simulator hit stop 1, breaking at the next instruction:
  0xb5355a04  e1a00000       mov r1, r1
sim> stop info all
Stop information:
stop 1 - 0x1:         Enabled,      counter = 1
stop 123 - 0x7b:      Enabled,      counter = 1
```

可以禁用或启用停靠点。（仅适用于已监视的停靠点。

```simulator
sim> stop disable 1
sim> cont
Simulator hit stop 123, breaking at the next instruction:
  0xb5356808  e1a00000       mov r0, r0
sim> cont
Simulator hit stop 123, breaking at the next instruction:
  0xb5356c28  e1a00000       mov r0, r0
sim> stop info all
Stop information:
stop 1 - 0x1:         Disabled,     counter = 2
stop 123 - 0x7b:      Enabled,      counter = 3
sim> stop enable 1
sim> cont
Simulator hit stop 1, breaking at the next instruction:
  0xb5356c44  e1a00000       mov r1, r1
sim> stop disable all
sim> con
```

#### `Debug()`（64 位臂）{ #arm64\_debug }

```cpp
MacroAssembler::Debug(const char* message, uint32_t code, Instr params = BREAK);
```

默认情况下，此指令是断点，但也能够启用和禁用跟踪，就像您使用[`trace`](#trace)命令。您还可以为其提供消息和代码作为标识符。

想象一下，我们正在处理这个V8 C++代码，它取自准备帧以调用JS函数的本机内置。

```cpp
int64_t bad_frame_pointer = -1L;  // Bad frame pointer, should fail if it is used.
__ Mov(x13, bad_frame_pointer);
__ Mov(x12, StackFrame::TypeToMarker(type));
__ Mov(x11, ExternalReference::Create(IsolateAddressId::kCEntryFPAddress,
                                      masm->isolate()));
__ Ldr(x10, MemOperand(x11));

__ Push(x13, x12, xzr, x10);
```

插入断点可能很有用`DebugBreak()`因此，我们可以在运行此函数时检查当前状态。但是，如果我们使用，我们可以更进一步并跟踪此代码`Debug()`相反：

```cpp
// Start tracing and log disassembly and register values.
__ Debug("start tracing", 42, TRACE_ENABLE | LOG_ALL);

int64_t bad_frame_pointer = -1L;  // Bad frame pointer, should fail if it is used.
__ Mov(x13, bad_frame_pointer);
__ Mov(x12, StackFrame::TypeToMarker(type));
__ Mov(x11, ExternalReference::Create(IsolateAddressId::kCEntryFPAddress,
                                      masm->isolate()));
__ Ldr(x10, MemOperand(x11));

__ Push(x13, x12, xzr, x10);

// Stop tracing.
__ Debug("stop tracing", 42, TRACE_DISABLE);
```

它允许我们跟踪寄存器值**只**我们正在处理的代码片段：

```simulator
$ d8 --allow-natives-syntax --debug-sim test.js
# NZCV: N:0 Z:0 C:0 V:0
# FPCR: AHP:0 DN:0 FZ:0 RMode:0b00 (Round to Nearest)
#    x0: 0x00007fbf00000000
#    x1: 0x00007fbf0804030d
#    x2: 0x00007fbf082500e1
(...)

0x00007fc039d31cb0  9280000d            movn x13, #0x0
#   x13: 0xffffffffffffffff
0x00007fc039d31cb4  d280004c            movz x12, #0x2
#   x12: 0x0000000000000002
0x00007fc039d31cb8  d2864110            movz x16, #0x3208
#   ip0: 0x0000000000003208
0x00007fc039d31cbc  8b10034b            add x11, x26, x16
#   x11: 0x00007fbf00003208
0x00007fc039d31cc0  f940016a            ldr x10, [x11]
#   x10: 0x0000000000000000 <- 0x00007fbf00003208
0x00007fc039d31cc4  a9be7fea            stp x10, xzr, [sp, #-32]!
#    sp: 0x00007fc033e81340
#   x10: 0x0000000000000000 -> 0x00007fc033e81340
#   xzr: 0x0000000000000000 -> 0x00007fc033e81348
0x00007fc039d31cc8  a90137ec            stp x12, x13, [sp, #16]
#   x12: 0x0000000000000002 -> 0x00007fc033e81350
#   x13: 0xffffffffffffffff -> 0x00007fc033e81358
0x00007fc039d31ccc  910063fd            add fp, sp, #0x18 (24)
#    fp: 0x00007fc033e81358
0x00007fc039d31cd0  d45bd600            hlt #0xdeb0
```
