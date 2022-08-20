***

## 标题： “在 Arm64 上编译”&#xA;描述： '在 Arm64 上原生构建 V8 的提示和技巧'

如果您已经浏览了有关如何操作的说明[退房](/docs/source-code)和[建](/docs/build-gn)在不是 x86 的机器上的 V8，您可能遇到了一些麻烦，因为构建系统下载了本机二进制文件，然后无法运行它们。但是，即使使用Arm64机器在V8上工作**不受官方支持**，克服这些障碍非常简单。

## 绕过`vpython`

`fetch v8`,`gclient sync`和其他`depot_tools`命令使用一个名为“vpython”的python包装器。如果看到与之相关的错误，可以定义以下变量以改用系统的 python 安装：

```bash
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

## 建筑`ninja`和`gn`从源头

首先要做的是确保我们有本地二进制文件`ninja`和`gn`，并且我们选择那些而不是`depot_tools`.执行此操作的一种简单方法是在安装时按如下方式调整 PATH`depot_tools`:

```bash
export PATH=$PATH:/path/to/depot_tools
```

这样，您将能够使用系统的`ninja`安装，因为它可能可用。虽然如果不是，你可以[从源代码构建它](https://github.com/ninja-build/ninja#building-ninja-itself).

然后，您将需要`gn`，可以使用以下命令构建：

```bash
git clone https://gn.googlesource.com/gn
cd gn
python build/gen.py
ninja -C out
```

最后调整你的`PATH`因此，它先于`gn`从`depot_tools`.

```bash
export PATH=/path/to/gn/out:$PATH
```

## 编译叮当

默认情况下，V8 希望使用自己的 clang 版本，该版本可能无法在您的机器上运行。您可以调整 GN 参数，以便[使用系统的叮当声或 GCC](#system_clang_gcc)，但是，您可能希望使用与上游相同的 clang，因为它将是最受支持的版本。

您可以直接从 V8 检出功能在本地构建它：

```bash
./tools/clang/scripts/build.py --without-android --without-fuchsia \
                               --gcc-toolchain=/usr --use-system-cmake \
                               --disable-asserts
```

## 手动设置 GN 参数

默认情况下，方便脚本可能不起作用，相反，您必须按照[手动](/docs/build-gn#gn)工作流。您可以使用以下参数获取通常的“release”，“optdebug”和“debug”配置：

*   `release`

```bash
is_debug=false
```

*   `optdebug`

```bash
is_debug=true
v8_enable_backtrace=true
v8_enable_slow_dchecks=true
```

*   `debug`

```bash
is_debug=true
v8_enable_backtrace=true
v8_enable_slow_dchecks=true
v8_optimized_debug=false
```

## 使用系统的 clang 或 GCC { #system_clang_gcc }

使用 GCC 进行构建只是禁用使用 clang 进行编译的一种情况：

```bash
is_clang=false
```

请注意，默认情况下，V8 将使用`lld`，这需要最新版本的 GCC。您可以使用`use_lld=false`切换到黄金链接器，或另外使用`use_gold=false`使用`ld`.

如果您想使用随系统一起安装的 clang，请输入`/usr`中，可以使用以下参数：

```bash
clang_base_path="/usr"
clang_use_chrome_plugins=false
```

但是，鉴于系统的 clang 版本可能没有得到很好的支持，您可能会处理警告，例如未知的编译器标志。在这种情况下，停止将警告视为错误非常有用：

```bash
treat_warnings_as_errors=false
```
