***

## 标题： '测试'&#xA;描述：“本文档解释了作为 V8 存储库一部分的测试框架。

V8 包含一个测试框架，允许您测试引擎。该框架允许您运行我们自己的测试套件（包含在源代码中）和其他测试套件，例如[Test262 测试套件](https://github.com/tc39/test262).

## 运行 V8 测试

[用`gm`](/docs/build-gn#gm)，您可以简单地追加`.check`到任何构建目标以使其运行测试，例如

```bash
gm x64.release.check
gm x64.optdebug.check  # recommended: reasonably fast, with DCHECKs.
gm ia32.check
gm release.check
gm check  # builds and tests all default platforms
```

`gm`在运行测试之前自动生成任何所需的目标。您还可以限制要运行的测试：

```bash
gm x64.release test262
gm x64.debug mjsunit/regress/regress-123
```

如果您已经构建了 V8，则可以手动运行测试：

```bash
tools/run-tests.py --outdir=out/ia32.release
```

同样，您可以指定要运行的测试：

```bash
tools/run-tests.py --outdir=ia32.release cctest/test-heap/SymbolTable/* mjsunit/delete-in-eval
```

运行脚本`--help`以了解其其他选项。

## 运行更多测试

要运行的默认测试集不包括所有可用的测试。您可以在以下任一命令的命令行上指定其他测试套件`gm`或`run-tests.py`:

*   `benchmarks`（只是为了正确性;不产生基准结果！
*   `mozilla`
*   `test262`
*   `webkit`

## 运行微基准标记

下`test/js-perf-test`我们有微基准标记来跟踪功能性能。有一个特殊的跑步者用于这些：`tools/run_perf.py`.像这样运行它们：

```bash
tools/run_perf.py --arch x64 --binary-override-path out/x64.release/d8 test/js-perf-test/JSTests.json
```

如果您不想运行所有`JSTests`，您可以提供一个`filter`论点：

```bash
tools/run_perf.py --arch x64 --binary-override-path out/x64.release/d8 --filter JSTests/TypedArrays test/js-perf-test/JSTests.json
```

## 更新检查员测试预期

更新测试后，您可能需要为其重新生成预期文件。您可以通过运行以下命令来实现此目的：

```bash
tools/run-tests.py --regenerate-expected-files --outdir=ia32.release inspector/debugger/set-instrumentation-breakpoint
```

如果您想了解测试的输出是如何变化的，这也很有用。首先使用上述命令重新生成预期的文件，然后通过以下方式检查差异：

```bash
git diff
```

## 更新字节码期望（重新基线）

有时，字节码期望值可能会发生变化，从而导致`cctest`失败。要更新黄金文件，请构建`test/cctest/generate-bytecode-expectations`通过运行：

```bash
gm x64.release generate-bytecode-expectations
```

...然后通过传递`--rebaseline`标志到生成的二进制文件：

```bash
out/x64.release/generate-bytecode-expectations --rebaseline
```

更新后的黄金现在可在`test/cctest/interpreter/bytecode_expectations/`.

## 添加新的字节码期望测试

1.  将新的测试用例添加到`cctest/interpreter/test-bytecode-generator.cc`并指定具有相同测试名称的黄金文件。

2.  建`generate-bytecode-expectations`:

    ```bash
    gm x64.release generate-bytecode-expectations
    ```

3.  跑

    ```bash
    out/x64.release/generate-bytecode-expectations --raw-js testcase.js --output=test/cctest/interpreter/bytecode-expectations/testname.golden
    ```

    哪里`testcase.js`包含已添加到`test-bytecode-generator.cc`和`testname`是 中定义的测试的名称`test-bytecode-generator.cc`.
