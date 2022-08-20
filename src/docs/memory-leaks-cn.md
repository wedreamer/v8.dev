***

## 标题： “调查内存泄漏”&#xA;描述：“本文档提供了有关调查 V8 中内存泄漏的指南。

如果您正在调查内存泄漏，并想知道为什么某个对象未被垃圾回收，则可以使用`%DebugTrackRetainingPath(object)`以在每个 GC 上打印对象的实际保留路径。

这需要`--allow-natives-syntax --track-retaining-path`运行时标志，并在发布和调试模式下工作。详细信息请参阅 CL 说明。

请考虑以下事项`test.js`:

```js
function foo() {
  const x = { bar: 'bar' };
  %DebugTrackRetainingPath(x);
  return () => { return x; }
}
const closure = foo();
gc();
```

示例（使用调试模式或`v8_enable_object_print = true`对于更详细的输出）：

```bash
$ out/x64.release/d8 --allow-natives-syntax --track-retaining-path --expose-gc test.js
#################################################
Retaining path for 0x245c59f0c1a1:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 6: 0x245c59f0c1a1 <Object map = 0x2d919f0d729>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 5: 0x245c59f0c169 <FixedArray[5]>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 4: 0x245c59f0c219 <JSFunction (sfi = 0x1fbb02e2d7f1)>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 3: 0x1fbb02e2d679 <FixedArray[5]>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 2: 0x245c59f0c139 <FixedArray[4]>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Distance from root 1: 0x1fbb02e03d91 <FixedArray[279]>

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Root: (Isolate)
-------------------------------------------------
```

## 调试器支持

在调试器会话中时（例如`gdb`/`lldb`），并假设您将上述标志传递给进程（即`--allow-natives-syntax --track-retaining-path`），您也许能够`print isolate->heap()->PrintRetainingPath(HeapObject*)`在感兴趣的对象上。
