***

## 标题： 'ARM/Android 的交叉编译和调试'&#xA;描述： “本文档介绍了如何针对 ARM/Android 交叉编译 V8，以及如何对其进行调试。

首先，确保您可以[使用 GN 构建](/docs/build-gn).

然后，添加`android`到您的`.gclient`配置文件。

```python
target_os = ['android']  # Add this to get Android stuff checked out.
```

这`target_os`字段是一个列表，所以如果你也在 unix 上构建，它看起来像这样：

```python
target_os = ['android', 'unix']  # Multiple target OSes.
```

跑`gclient sync`，您将在下获得一个大结帐`./third_party/android_tools`.

在手机或平板电脑上启用开发人员模式，并通过说明打开USB调试[这里](https://developer.android.com/studio/run/device.html).另外，得到方便[`adb`](https://developer.android.com/studio/command-line/adb.html)工具在您的路径上。它位于您的结帐处`./third_party/android_sdk/public/platform-tools`.

## 用`gm`

用[这`tools/dev/gm.py`脚本](/docs/build-gn#gm)以自动构建 V8 测试并在设备上运行它们。

```bash
alias gm=/path/to/v8/tools/dev/gm.py
gm android_arm.release.check
```

此命令将二进制文件和测试推送到`/data/local/tmp/v8`目录。

## 手动构建

用`v8gen.py`生成 ARM 版本或调试版本：

```bash
tools/dev/v8gen.py arm.release
```

然后运行`gn args out.gn/arm.release`，并确保您具有以下键：

```python
target_os = "android"      # These lines need to be changed manually
target_cpu = "arm"         # as v8gen.py assumes a simulator build.
v8_target_cpu = "arm"
is_component_build = false
```

调试版本的键应相同。如果您要为支持 32 位和 64 位二进制文件的 arm64 设备（如 Pixel C）进行构建，则密钥应如下所示：

```python
target_os = "android"      # These lines need to be changed manually
target_cpu = "arm64"       # as v8gen.py assumes a simulator build.
v8_target_cpu = "arm64"
is_component_build = false
```

现在构建：

```bash
ninja -C out.gn/arm.release d8
```

用`adb`将二进制文件和快照文件复制到手机：

```bash
adb shell 'mkdir -p /data/local/tmp/v8/bin'
adb push out.gn/arm.release/d8 /data/local/tmp/v8/bin
adb push out.gn/arm.release/icudtl.dat /data/local/tmp/v8/bin
adb push out.gn/arm.release/snapshot_blob.bin /data/local/tmp/v8/bin
```

```bash
rebuffat:~/src/v8$ adb shell
bullhead:/ $ cd /data/local/tmp/v8/bin
bullhead:/data/local/tmp/v8/bin $ ls
v8 icudtl.dat snapshot_blob.bin
bullhead:/data/local/tmp/v8/bin $ ./d8
V8 version 5.8.0 (candidate)
d8> 'w00t!'
"w00t!"
d8>
```

## 调试

### d8

远程调试`d8`在安卓设备上相对简单。首次启动`gdbserver`在安卓设备上：

```bash
bullhead:/data/local/tmp/v8/bin $ gdbserver :5039 $D8 <arguments>
```

然后连接到主机设备上的服务器。

```bash
adb forward tcp:5039 tcp:5039
gdb $D8
gdb> target remote :5039
```

`gdb`和`gdbserver`需要相互兼容，如有疑问，请使用来自[安卓 NDK](https://developer.android.com/ndk).请注意，默认情况下`d8`二进制文件被剥离（调试信息已删除），`$OUT_DIR/exe.unstripped/d8`包含未剥离的二进制文件。

### 伐木

默认情况下，某些`d8`的调试输出最终出现在 Android 系统日志中，可以使用[`logcat`](https://developer.android.com/studio/command-line/logcat).遗憾的是，有时特定调试输出的一部分会在系统日志和`adb`，有时某些部分似乎完全缺失。为避免这些问题，建议将以下设置添加到`gn args`:

```python
v8_android_log_stdout = true
```

### 浮点问题

这`gn args`设置`arm_float_abi = "hard"`，由 V8 Arm GC Stress bot 使用，可能会导致硬件上完全荒谬的程序行为，这与 GC 压力机器人使用的硬件不同（例如，在 Nexus 7 上）。

## 使用 Sourcery G++ Lite

Sourcery G++ Lite 交叉编译器套件是 Sourcery G++ 的免费版本，来自[代码库](http://www.codesourcery.com/).有一个页面[用于 ARM 处理器的 GNU 工具链](http://www.codesourcery.com/sgpp/lite/arm).确定主机/目标组合所需的版本。

以下说明使用[2009q1-203 for ARM GNU/Linux](http://www.codesourcery.com/sgpp/lite/arm/portal/release858)，如果使用其他版本，请更改 URL 和`TOOL_PREFIX`相应地在下面。

### 在主机和目标上安装

设置此项的最简单方法是在同一位置的主机和目标上安装完整的 Sourcery G++ Lite 包。这将确保所有所需的库在两端都可用。如果要在主机上使用默认库，则无需在目标上安装任何内容。

以下脚本安装在`/opt/codesourcery`:

```bash
#!/bin/sh

sudo mkdir /opt/codesourcery
cd /opt/codesourcery
sudo chown "$USERNAME" .
chmod g+ws .
umask 2
wget http://www.codesourcery.com/sgpp/lite/arm/portal/package4571/public/arm-none-linux-gnueabi/arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2
tar -xvf arm-2009q1-203-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2
```

## 轮廓

*   编译二进制文件，将其推送到设备，在主机上保留其副本：

    ```bash
    adb shell cp /data/local/tmp/v8/bin/d8 /data/local/tmp/v8/bin/d8-version.under.test
    cp out.gn/arm.release/d8 ./d8-version.under.test
    ```

*   获取分析日志并将其复制到主机：

    ```bash
    adb push benchmarks /data/local/tmp
    adb shell cd /data/local/tmp/benchmarks; ../v8/bin/d8-version.under.test run.js --prof
    adb shell /data/local/tmp/v8/bin/d8-version.under.test benchmark.js --prof
    adb pull /data/local/tmp/benchmarks/v8.log ./
    ```

*   打开`v8.log`在您喜欢的编辑器中，然后编辑第一行以匹配的完整路径`d8-version.under.test`工作站上的二进制文件（而不是`/data/local/tmp/v8/bin/`它在设备上的路径）

*   使用主机的`d8`和适当的`nm`二元的：

    ```bash
    cp out/x64.release/d8 .  # only required once
    cp out/x64.release/natives_blob.bin .  # only required once
    cp out/x64.release/snapshot_blob.bin .  # only required once
    tools/linux-tick-processor --nm=$(pwd)/third_party/android_ndk/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/arm-linux-androideabi-nm
    ```
