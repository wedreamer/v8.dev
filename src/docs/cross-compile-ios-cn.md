***

## 标题： '针对 iOS 的交叉编译'&#xA;描述：“本文档介绍如何交叉编译适用于 iOS 的 V8。

本页简要介绍了如何为 iOS 目标构建 V8。

## 要求

*   安装了 Xcode 的 macOS （OS X） 主机。
*   64 位目标 iOS 设备（不支持旧版 32 位 iOS 设备）。
*   V8 v7.5 或更高版本。
*   jitless是iOS的硬性要求（截至2020年12月）。因此，请使用标志“--expose_gc--jitless”

## 初始设置

跟随[构建 V8 的说明](/docs/build).

通过添加`target_os`在您的`.gclient`配置文件，位于`v8`源目录：

```python
# [... other contents of .gclient such as the 'solutions' variable ...]
target_os = ['ios']
```

更新后`.gclient`跑`gclient sync`以下载其他工具。

## 手动构建

本节介绍如何构建在物理 iOS 设备或 Xcode iOS 模拟器上使用的整体式 V8 版本。此版本的输出是`libv8_monolith.a`文件，其中包含所有 V8 库以及 V8 快照。

通过运行以下命令设置 GN 构建文件`gn args out/release-ios`并插入以下键：

```python
enable_ios_bitcode = true
ios_deployment_target = 10
is_component_build = false
is_debug = false
target_cpu = "arm64"                  # "x64" for a simulator build.
target_os = "ios"
use_custom_libcxx = false             # Use Xcode's libcxx.
use_xcode_clang = true
v8_enable_i18n_support = false        # Produces a smaller binary.
v8_monolithic = true                  # Enable the v8_monolith target.
v8_use_external_startup_data = false  # The snaphot is included in the binary.
v8_enable_pointer_compression = false # Unsupported on iOS.
```

现在构建：

```bash
ninja -C out/release-ios v8_monolith
```

最后，添加生成的`libv8_monolith.a`文件作为静态库添加到 Xcode 项目中。有关在应用程序中嵌入 V8 的更多文档，请参阅[嵌入 V8 入门](/docs/embed).
