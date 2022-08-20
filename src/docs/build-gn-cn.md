***

## 标题： “用GN建造V8”&#xA;描述： “本文档介绍如何使用 GN 构建 V8。

V8 是在以下方面的帮助下构建的[断续器](https://gn.googlesource.com/gn/+/master/docs/).GN是一个元构建系统，因为它为许多其他构建系统生成构建文件。因此，如何构建取决于您使用的“后端”构建系统和编译器。
以下说明假定您已经有一个[检出 V8](/docs/source-code)并且您有[已安装构建依赖项](/docs/build).

有关 GN 的更多信息，请参见[铬的文档](https://www.chromium.org/developers/gn-build-configuration)或[GN 自己的文档](https://gn.googlesource.com/gn/+/master/docs/).

从源代码构建 V8 包括三个步骤：

1.  生成构建文件
2.  编译
3.  运行测试

构建 V8 有两个工作流：

*   使用名为`gm`很好地结合了所有三个步骤
*   原始工作流，您可以在其中为每个步骤在较低级别手动运行单独的命令

## 构建 V8 使用`gm`（方便的工作流程）{ #gm }

`gm`是一个方便的一体化脚本，可生成生成文件，触发生成并选择性地运行测试。它可以在以下位置找到`tools/dev/gm.py`在您的 V8 结账中。我们建议在 shell 配置中添加别名：

```bash
alias gm=/path/to/v8/tools/dev/gm.py
```

然后，您可以使用`gm`为已知配置构建 V8，例如`x64.release`:

```bash
gm x64.release
```

若要在生成后立即运行测试，请运行：

```bash
gm x64.release.check
```

`gm`输出它正在执行的所有命令，以便于跟踪并在必要时重新执行它们。

`gm`允许使用单个命令构建所需的二进制文件并运行特定测试：

```bash
gm x64.debug mjsunit/foo cctest/test-bar/*
```

## 构建 V8：原始的手动工作流 { #manual }

### 步骤 1：生成构建文件 { #generate构建文件 }

有几种方法可以生成生成文件：

1.  原始的手动工作流程包括使用`gn`径直。
2.  名为 的帮助程序脚本`v8gen`简化了常见配置的过程。

#### 使用生成构建文件`gn`{ #gn }

为目录生成构建文件`out/foo`用`gn`:

```bash
gn args out/foo
```

这将打开一个编辑器窗口，用于指定[`gn`参数](https://gn.googlesource.com/gn/+/master/docs/reference.md).或者，您可以在命令行上传递参数：

```bash
gn gen out/foo --args='is_debug=false target_cpu="x64" v8_target_cpu="arm64" use_goma=true'
```

这将生成构建文件，用于在发布模式下使用 arm64 模拟器编译 V8，使用`goma`用于编译。

有关所有可用选项的概述`gn`参数， 运行：

```bash
gn args out/foo --list
```

#### 使用生成构建文件`v8gen`{ #v8gen }

V8 存储库包括一个`v8gen`方便的脚本，可以更轻松地为常见配置生成构建文件。我们建议在 shell 配置中添加别名：

```bash
alias v8gen=/path/to/v8/tools/dev/v8gen.py
```

叫`v8gen --help`了解更多信息。

列出可用配置（或来自主节点的机器人）：

```bash
v8gen list
```

```bash
v8gen list -m client.v8
```

像特定机器人一样从`client.v8`文件夹中的瀑布`foo`:

```bash
v8gen -b 'V8 Linux64 - debug builder' -m client.v8 foo
```

### 步骤 2：编译 V8 { #compile }

构建所有 V8（假设`gn`生成到`x64.release`文件夹），运行：

```bash
ninja -C out/x64.release
```

构建特定目标，例如`d8`，将它们附加到命令中：

```bash
ninja -C out/x64.release d8
```

### 步骤 3：运行测试 { #tests }

可以将输出目录传递给测试驱动程序。其他相关标志是从构建中推断出来的：

```bash
tools/run-tests.py --outdir out/foo
```

您还可以测试最近编译的生成（在`out.gn`):

```bash
tools/run-tests.py --gn
```

**构建问题？在 上提交错误[v8.dev/bug](/bug)或寻求帮助<v8-users@googlegroups.com>.**
