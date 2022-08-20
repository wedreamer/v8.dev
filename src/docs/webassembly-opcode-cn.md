***

## 标题：“WebAssembly - 添加新的操作码”&#xA;描述： '本教程解释了如何在 V8 中实现新的 WebAssembly 指令。

[WebAssembly](https://webassembly.org/)（Wasm） 是基于堆栈的虚拟机的二进制指令格式。本教程将引导读者在 V8 中实现新的 WebAssembly 指令。

WebAssembly 在 V8 中分三部分实现：

*   口译员
*   基线编译器（Liftoff）
*   优化编译器（TurboFan）

本文档的其余部分重点介绍 TurboFan 流水线，介绍如何添加新的 Wasm 指令并在 TurboFan 中实现它。

在较高层次上，Wasm指令被编译成TurboFan图，我们依靠TurboFan管道将图编译成（最终）机器代码。有关 TurboFan 的更多信息，请查看[V8 文档](/docs/turbofan).

## 操作码/说明

让我们定义一个新指令，该指令将`1`到[`int32`](https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype)（在堆栈的顶部）。

：：：备注
**注意：**所有 Wasm 实现都支持的说明列表可以在[规范](https://webassembly.github.io/spec/core/appendix/index-instructions.html).
:::

所有 Wasm 指令都定义在[`src/wasm/wasm-opcodes.h`](https://cs.chromium.org/chromium/src/v8/src/wasm/wasm-opcodes.h).指令大致按其作用进行分组，例如控制，内存，SIMD，原子等。

让我们添加我们的新指令，`I32Add1`，则`FOREACH_SIMPLE_OPCODE`部分：

```diff
diff --git a/src/wasm/wasm-opcodes.h b/src/wasm/wasm-opcodes.h
index 6970c667e7..867cbf451a 100644
--- a/src/wasm/wasm-opcodes.h
+++ b/src/wasm/wasm-opcodes.h
@@ -96,6 +96,7 @@ bool IsJSCompatibleSignature(const FunctionSig* sig, bool hasBigIntFeature);

 // Expressions with signatures.
 #define FOREACH_SIMPLE_OPCODE(V)  \
+  V(I32Add1, 0xee, i_i)           \
   V(I32Eqz, 0x45, i_i)            \
   V(I32Eq, 0x46, i_ii)            \
   V(I32Ne, 0x47, i_ii)            \
```

WebAssembly是一种二进制格式，所以`0xee`指定此指令的编码。在本教程中，我们选择了`0xee`因为它当前未使用。

：：：备注
**注意：**实际上，向规范中添加指令涉及超出此处描述范围的工作。
:::

我们可以使用以下命令对操作码运行一个简单的单元测试：

    $ tools/dev/gm.py x64.debug unittests/WasmOpcodesTest*
    ...
    [==========] Running 1 test from 1 test suite.
    [----------] Global test environment set-up.
    [----------] 1 test from WasmOpcodesTest
    [ RUN      ] WasmOpcodesTest.EveryOpcodeHasAName
    ../../test/unittests/wasm/wasm-opcodes-unittest.cc:27: Failure
    Value of: false
      Actual: false
    Expected: true
    WasmOpcodes::OpcodeName(kExprI32Add1) == "unknown"; plazz halp in src/wasm/wasm-opcodes.cc
    [  FAILED  ] WasmOpcodesTest.EveryOpcodeHasAName

此错误表示我们没有新指令的名称。可以为新操作码添加名称，可在[`src/wasm/wasm-opcodes.cc`](https://cs.chromium.org/chromium/src/v8/src/wasm/wasm-opcodes.cc):

```diff
diff --git a/src/wasm/wasm-opcodes.cc b/src/wasm/wasm-opcodes.cc
index 5ed664441d..2d4e9554fe 100644
--- a/src/wasm/wasm-opcodes.cc
+++ b/src/wasm/wasm-opcodes.cc
@@ -75,6 +75,7 @@ const char* WasmOpcodes::OpcodeName(WasmOpcode opcode) {
     // clang-format off

     // Standard opcodes
+    CASE_I32_OP(Add1, "add1")
     CASE_INT_OP(Eqz, "eqz")
     CASE_ALL_OP(Eq, "eq")
     CASE_I64x2_OP(Eq, "eq")
```

通过在`FOREACH_SIMPLE_OPCODE`，我们正在跳过一个[相当多的工作量](https://cs.chromium.org/chromium/src/v8/src/wasm/function-body-decoder-impl.h?l=1751-1756\&rcl=686b68edf9f42c201c2b25bca9f4bef72ff41c0b)在`src/wasm/function-body-decoder-impl.h`，它解码 Wasm 操作码并调用 TurboFan 图形生成器。因此，根据您的操作码的作用，您可能还有更多工作要做。为简洁起见，我们跳过此步骤。

## 为新操作码编写测试 { #test }

Wasm 测试可以在[`test/cctest/wasm/`](https://cs.chromium.org/chromium/src/v8/test/cctest/wasm/).让我们来看看[`test/cctest/wasm/test-run-wasm.cc`](https://cs.chromium.org/chromium/src/v8/test/cctest/wasm/test-run-wasm.cc)，其中测试了许多“简单”操作码。

此文件中有许多我们可以遵循的示例。常规设置是：

*   创建一个`WasmRunner`
*   设置全局变量以保存结果（可选）
*   将局部变量设置为指令的参数（可选）
*   构建 wasm 模块
*   运行它并与预期输出进行比较

以下是我们新操作码的简单测试：

```diff
diff --git a/test/cctest/wasm/test-run-wasm.cc b/test/cctest/wasm/test-run-wasm.cc
index 26df61ceb8..b1ee6edd71 100644
--- a/test/cctest/wasm/test-run-wasm.cc
+++ b/test/cctest/wasm/test-run-wasm.cc
@@ -28,6 +28,15 @@ namespace test_run_wasm {
 #define RET(x) x, kExprReturn
 #define RET_I8(x) WASM_I32V_2(x), kExprReturn

+#define WASM_I32_ADD1(x) x, kExprI32Add1
+
+WASM_EXEC_TEST(Int32Add1) {
+  WasmRunner<int32_t> r(execution_tier);
+  // 10 + 1
+  BUILD(r, WASM_I32_ADD1(WASM_I32V_1(10)));
+  CHECK_EQ(11, r.Call());
+}
+
 WASM_EXEC_TEST(Int32Const) {
   WasmRunner<int32_t> r(execution_tier);
   const int32_t kExpectedValue = 0x11223344;
```

运行测试：

    $ tools/dev/gm.py x64.debug 'cctest/test-run-wasm-simd/RunWasmTurbofan_I32Add1'
    ...
    === cctest/test-run-wasm/RunWasmTurbofan_Int32Add1 ===
    #
    # Fatal error in ../../src/compiler/wasm-compiler.cc, line 988
    # Unsupported opcode 0xee:i32.add1

：：：备注
**提示：**查找测试名称可能很棘手，因为测试定义位于宏后面。用[代码搜索](https://cs.chromium.org/)以单击以发现宏定义。
:::

此错误表示编译器不知道我们的新指令。这将在下一节中发生变化。

## 将 Wasm 编译为 TurboFan

在引言中，我们提到 Wasm 指令被编译成 TurboFan 图。`wasm-compiler.cc`是发生这种情况的地方。让我们看一个示例操作码，[`I32Eqz`](https://cs.chromium.org/chromium/src/v8/src/compiler/wasm-compiler.cc?l=716\&rcl=686b68edf9f42c201c2b25bca9f4bef72ff41c0b):

```cpp
  switch (opcode) {
    case wasm::kExprI32Eqz:
      op = m->Word32Equal();
      return graph()->NewNode(op, input, mcgraph()->Int32Constant(0));
```

这将打开 Wasm 操作码`wasm::kExprI32Eqz`，并构建一个由操作组成的 TurboFan 图`Word32Equal`与输入`input`，这是 Wasm 指令的参数，也是一个常量`0`.

这`Word32Equal`运算符由底层 V8 抽象机提供，该抽象机独立于体系结构。稍后在管道中，此抽象机器运算符将转换为依赖于体系结构的程序集。

对于我们的新操作码，`I32Add1`，我们需要一个在输入中添加常量 1 的图形，以便我们可以重用现有的机器操作员，`Int32Add`，将它传递给输入，并得到一个常量 1：

```diff
diff --git a/src/compiler/wasm-compiler.cc b/src/compiler/wasm-compiler.cc
index f666bbb7c1..399293c03b 100644
--- a/src/compiler/wasm-compiler.cc
+++ b/src/compiler/wasm-compiler.cc
@@ -713,6 +713,8 @@ Node* WasmGraphBuilder::Unop(wasm::WasmOpcode opcode, Node* input,
   const Operator* op;
   MachineOperatorBuilder* m = mcgraph()->machine();
   switch (opcode) {
+    case wasm::kExprI32Add1:
+      return graph()->NewNode(m->Int32Add(), input, mcgraph()->Int32Constant(1));
     case wasm::kExprI32Eqz:
       op = m->Word32Equal();
       return graph()->NewNode(op, input, mcgraph()->Int32Constant(0));
```

这足以使测试通过。但是，并非所有指令都有现有的涡轮风扇机器操作员。在这种情况下，我们必须将此新操作员添加到机器中。让我们试试吧。

## 涡轮风扇机器操作员

我们想添加的知识`Int32Add1`到涡轮风扇机。因此，让我们假装它存在并首先使用它：

```diff
diff --git a/src/compiler/wasm-compiler.cc b/src/compiler/wasm-compiler.cc
index f666bbb7c1..1d93601584 100644
--- a/src/compiler/wasm-compiler.cc
+++ b/src/compiler/wasm-compiler.cc
@@ -713,6 +713,8 @@ Node* WasmGraphBuilder::Unop(wasm::WasmOpcode opcode, Node* input,
   const Operator* op;
   MachineOperatorBuilder* m = mcgraph()->machine();
   switch (opcode) {
+    case wasm::kExprI32Add1:
+      return graph()->NewNode(m->Int32Add1(), input);
     case wasm::kExprI32Eqz:
       op = m->Word32Equal();
       return graph()->NewNode(op, input, mcgraph()->Int32Constant(0));
```

尝试运行相同的测试会导致编译失败，提示在何处进行更改：

    ../../src/compiler/wasm-compiler.cc:717:34: error: no member named 'Int32Add1' in 'v8::internal::compiler::MachineOperatorBuilder'; did you mean 'Int32Add'?
          return graph()->NewNode(m->Int32Add1(), input);
                                     ^~~~~~~~~
                                     Int32Add

需要修改几个位置才能添加运算符：

1.  [`src/compiler/machine-operator.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/machine-operator.cc)
2.  页眉[`src/compiler/machine-operator.h`](https://cs.chromium.org/chromium/src/v8/src/compiler/machine-operator.h)
3.  机器理解的操作码列表[`src/compiler/opcodes.h`](https://cs.chromium.org/chromium/src/v8/src/compiler/opcodes.h)
4.  验证[`src/compiler/verifier.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/verifier.cc)

```diff
diff --git a/src/compiler/machine-operator.cc b/src/compiler/machine-operator.cc
index 16e838c2aa..fdd6d951f0 100644
--- a/src/compiler/machine-operator.cc
+++ b/src/compiler/machine-operator.cc
@@ -136,6 +136,7 @@ MachineType AtomicOpType(Operator const* op) {
 #define MACHINE_PURE_OP_LIST(V)                                               \
   PURE_BINARY_OP_LIST_32(V)                                                   \
   PURE_BINARY_OP_LIST_64(V)                                                   \
+  V(Int32Add1, Operator::kNoProperties, 1, 0, 1)                              \
   V(Word32Clz, Operator::kNoProperties, 1, 0, 1)                              \
   V(Word64Clz, Operator::kNoProperties, 1, 0, 1)                              \
   V(Word32ReverseBytes, Operator::kNoProperties, 1, 0, 1)                     \
```

```diff
diff --git a/src/compiler/machine-operator.h b/src/compiler/machine-operator.h
index a2b9fce0ee..f95e75a445 100644
--- a/src/compiler/machine-operator.h
+++ b/src/compiler/machine-operator.h
@@ -265,6 +265,8 @@ class V8_EXPORT_PRIVATE MachineOperatorBuilder final
   const Operator* Word32PairShr();
   const Operator* Word32PairSar();

+  const Operator* Int32Add1();
+
   const Operator* Int32Add();
   const Operator* Int32AddWithOverflow();
   const Operator* Int32Sub();
```

```diff
diff --git a/src/compiler/opcodes.h b/src/compiler/opcodes.h
index ce24a0bd3f..2c8c5ebaca 100644
--- a/src/compiler/opcodes.h
+++ b/src/compiler/opcodes.h
@@ -506,6 +506,7 @@
   V(Float64LessThanOrEqual)

 #define MACHINE_UNOP_32_LIST(V) \
+  V(Int32Add1)                  \
   V(Word32Clz)                  \
   V(Word32Ctz)                  \
   V(Int32AbsWithOverflow)       \
```

```diff
diff --git a/src/compiler/verifier.cc b/src/compiler/verifier.cc
index 461aef0023..95251934ce 100644
--- a/src/compiler/verifier.cc
+++ b/src/compiler/verifier.cc
@@ -1861,6 +1861,7 @@ void Verifier::Visitor::Check(Node* node, const AllNodes& all) {
     case IrOpcode::kSignExtendWord16ToInt64:
     case IrOpcode::kSignExtendWord32ToInt64:
     case IrOpcode::kStaticAssert:
+    case IrOpcode::kInt32Add1:

 #define SIMD_MACHINE_OP_CASE(Name) case IrOpcode::k##Name:
       MACHINE_SIMD_OP_LIST(SIMD_MACHINE_OP_CASE)
```

现在再次运行测试会给我们一个不同的失败：

    === cctest/test-run-wasm/RunWasmTurbofan_Int32Add1 ===
    #
    # Fatal error in ../../src/compiler/backend/instruction-selector.cc, line 2072
    # Unexpected operator #289:Int32Add1 @ node #7

## 指令选择

到目前为止，我们一直在TurboFan级别工作，处理TurboFan图中的（海量）节点。但是，在程序集级别，我们有指令和操作数。指令选择是将此图转换为指令和操作数的过程。

最后一个测试错误表明我们需要一些东西[`src/compiler/backend/instruction-selector.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/backend/instruction-selector.cc). 这是一个大文件，其中包含对所有机器操作码的巨型开关语句。 它调用特定于体系结构的指令选择，使用访问者模式为每个类型的节点发出指令。

由于我们添加了一个新的TurboFan机器操作码，因此我们也需要在此处添加它：

```diff
diff --git a/src/compiler/backend/instruction-selector.cc b/src/compiler/backend/instruction-selector.cc
index 3152b2d41e..7375085649 100644
--- a/src/compiler/backend/instruction-selector.cc
+++ b/src/compiler/backend/instruction-selector.cc
@@ -2067,6 +2067,8 @@ void InstructionSelector::VisitNode(Node* node) {
       return MarkAsWord32(node), VisitS1x16AnyTrue(node);
     case IrOpcode::kS1x16AllTrue:
       return MarkAsWord32(node), VisitS1x16AllTrue(node);
+    case IrOpcode::kInt32Add1:
+      return MarkAsWord32(node), VisitInt32Add1(node);
     default:
       FATAL("Unexpected operator #%d:%s @ node #%d", node->opcode(),
             node->op()->mnemonic(), node->id());
```

指令选择取决于架构，因此我们也必须将其添加到特定于架构的指令选择器文件中。对于这个代码实验室，我们只关注x64架构，所以[`src/compiler/backend/x64/instruction-selector-x64.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/backend/x64/instruction-selector-x64.cc)
需要修改：

```diff
diff --git a/src/compiler/backend/x64/instruction-selector-x64.cc b/src/compiler/backend/x64/instruction-selector-x64.cc
index 2324e119a6..4b55671243 100644
--- a/src/compiler/backend/x64/instruction-selector-x64.cc
+++ b/src/compiler/backend/x64/instruction-selector-x64.cc
@@ -841,6 +841,11 @@ void InstructionSelector::VisitWord32ReverseBytes(Node* node) {
   Emit(kX64Bswap32, g.DefineSameAsFirst(node), g.UseRegister(node->InputAt(0)));
 }

+void InstructionSelector::VisitInt32Add1(Node* node) {
+  X64OperandGenerator g(this);
+  Emit(kX64Int32Add1, g.DefineSameAsFirst(node), g.UseRegister(node->InputAt(0)));
+}
+
```

我们还需要添加这个新的特定于x64的操作码，`kX64Int32Add1`自[`src/compiler/backend/x64/instruction-codes-x64.h`](https://cs.chromium.org/chromium/src/v8/src/compiler/backend/x64/instruction-codes-x64.h):

```diff
diff --git a/src/compiler/backend/x64/instruction-codes-x64.h b/src/compiler/backend/x64/instruction-codes-x64.h
index 9b8be0e0b5..7f5faeb87b 100644
--- a/src/compiler/backend/x64/instruction-codes-x64.h
+++ b/src/compiler/backend/x64/instruction-codes-x64.h
@@ -12,6 +12,7 @@ namespace compiler {
 // X64-specific opcodes that specify which assembly sequence to emit.
 // Most opcodes specify a single instruction.
 #define TARGET_ARCH_OPCODE_LIST(V)        \
+  V(X64Int32Add1)                         \
   V(X64Add)                               \
   V(X64Add32)                             \
   V(X64And)                               \
```

## 指令调度和代码生成

运行我们的测试，我们看到新的编译错误：

    ../../src/compiler/backend/x64/instruction-scheduler-x64.cc:15:11: error: enumeration value 'kX64Int32Add1' not handled in switch [-Werror,-Wswitch]
      switch (instr->arch_opcode()) {
              ^
    1 error generated.
    ...
    ../../src/compiler/backend/x64/code-generator-x64.cc:733:11: error: enumeration value 'kX64Int32Add1' not handled in switch [-Werror,-Wswitch]
      switch (arch_opcode) {
              ^
    1 error generated.

[指令调度](https://en.wikipedia.org/wiki/Instruction_scheduling)处理指令可能必须允许更多优化的依赖关系（例如，指令重新排序）。我们的新操作码没有数据依赖关系，因此我们可以简单地将其添加到：[`src/compiler/backend/x64/instruction-scheduler-x64.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/backend/x64/instruction-scheduler-x64.cc):

```diff
diff --git a/src/compiler/backend/x64/instruction-scheduler-x64.cc b/src/compiler/backend/x64/instruction-scheduler-x64.cc
index 79eda7e78d..3667a84577 100644
--- a/src/compiler/backend/x64/instruction-scheduler-x64.cc
+++ b/src/compiler/backend/x64/instruction-scheduler-x64.cc
@@ -13,6 +13,7 @@ bool InstructionScheduler::SchedulerSupported() { return true; }
 int InstructionScheduler::GetTargetInstructionFlags(
     const Instruction* instr) const {
   switch (instr->arch_opcode()) {
+    case kX64Int32Add1:
     case kX64Add:
     case kX64Add32:
     case kX64And:
```

代码生成是我们将特定于体系结构的操作码转换为程序集的地方。让我们添加一个子句[`src/compiler/backend/x64/code-generator-x64.cc`](https://cs.chromium.org/chromium/src/v8/src/compiler/backend/x64/code-generator-x64.cc):

```diff
diff --git a/src/compiler/backend/x64/code-generator-x64.cc b/src/compiler/backend/x64/code-generator-x64.cc
index 61c3a45a16..9c37ed7464 100644
--- a/src/compiler/backend/x64/code-generator-x64.cc
+++ b/src/compiler/backend/x64/code-generator-x64.cc
@@ -731,6 +731,9 @@ CodeGenerator::CodeGenResult CodeGenerator::AssembleArchInstruction(
   InstructionCode opcode = instr->opcode();
   ArchOpcode arch_opcode = ArchOpcodeField::decode(opcode);
   switch (arch_opcode) {
+    case kX64Int32Add1: {
+      break;
+    }
     case kArchCallCodeObject: {
       if (HasImmediateInput(instr, 0)) {
         Handle<Code> code = i.InputCode(0);
```

现在，我们将代码生成留空，我们可以运行测试以确保所有内容都编译：

    === cctest/test-run-wasm/RunWasmTurbofan_Int32Add1 ===
    #
    # Fatal error in ../../test/cctest/wasm/test-run-wasm.cc, line 37
    # Check failed: 11 == r.Call() (11 vs. 10).

这种失败是意料之中的，因为我们的新指令尚未实现 - 它本质上是一个no-op，因此我们的实际值保持不变（`10`).

要实现我们的操作码，我们可以使用`add`组装说明：

```diff
diff --git a/src/compiler/backend/x64/code-generator-x64.cc b/src/compiler/backend/x64/code-generator-x64.cc
index 6c828d6bc4..260c8619f2 100644
--- a/src/compiler/backend/x64/code-generator-x64.cc
+++ b/src/compiler/backend/x64/code-generator-x64.cc
@@ -744,6 +744,11 @@ CodeGenerator::CodeGenResult CodeGenerator::AssembleArchInstruction(
   InstructionCode opcode = instr->opcode();
   ArchOpcode arch_opcode = ArchOpcodeField::decode(opcode);
   switch (arch_opcode) {
+    case kX64Int32Add1: {
+      DCHECK_EQ(i.OutputRegister(), i.InputRegister(0));
+      __ addl(i.InputRegister(0), Immediate(1));
+      break;
+    }
     case kArchCallCodeObject: {
       if (HasImmediateInput(instr, 0)) {
         Handle<Code> code = i.InputCode(0);
```

这使测试通过：

对我们来说很幸运`addl`已实现。如果我们的新操作码需要编写新的汇编指令实现，我们会将其添加到[`src/compiler/backend/x64/assembler-x64.cc`](https://cs.chromium.org/chromium/src/v8/src/codegen/x64/assembler-x64.cc)，其中汇编指令被编码为字节并发出。

：：：备注
**提示：**要检查生成的代码，我们可以通过`--print-code`自`cctest`.
:::

## 其他体系结构

在这个代码实验室中，我们只为x64实现了这个新指令。其他架构所需的步骤类似：添加TurboFan机器操作员，使用与平台相关的文件进行指令选择，调度，代码生成，汇编程序。

提示：如果我们在另一个目标（例如arm64）上编译到目前为止所做的工作，我们可能会在链接中遇到错误。要解决这些错误，请添加`UNIMPLEMENTED()`存根。
