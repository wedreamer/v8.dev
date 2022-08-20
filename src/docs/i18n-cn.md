***

## 标题： 'i18n 支持'&#xA;描述： 'V8 可以选择性地实现 ECMAScript 402 规范。默认情况下，API处于启用状态，但可以在编译时关闭。

V8 可选择实现[ECMAScript 402 规范](https://tc39.es/ecma402/).默认情况下，该 API 处于启用状态，但可以在编译时关闭。

## 先决条件

i18n 实现增加了对 ICU 的依赖性。从 v7.2 开始，V8 至少需要 ICU 版本 63。确切的依赖关系在[V8的`DEPS`文件](https://chromium.googlesource.com/v8/v8.git/+/master/DEPS).

运行以下命令以将合适版本的 ICU 检出到`third_party/icu`:

```bash
gclient sync
```

看[“保持最新状态”](/docs/source-code#staying-up-to-date)了解更多详情。

## 替代 ICU 结账

您可以在其他位置签出 ICU 源并定义 gyp 变量`icu_gyp_path`指向`icu.gyp`文件。

## 系统重症监护室

最后但并非最不重要的一点是，您可以针对系统中安装的 ICU 版本编译 V8。为此，请指定 GYP 变量`use_system_icu=1`.如果您还有`want_separate_host_toolset`启用后，仍将编译捆绑的 ICU 以生成 V8 快照。系统 ICU 仅用于目标体系结构。

## 嵌入 V8

如果在应用程序中嵌入了 V8，但应用程序本身不使用 ICU，则需要在调用 V8 之前通过执行以下命令来初始化 ICU：

```cpp
v8::V8::InitializeICU();
```

如果未编译 ICU，则调用此方法是安全的，然后它不执行任何操作。

## 在不支持 i18n 的情况下进行编译

要在没有 i18n 支持的情况下构建 V8，请使用[`gn args`](/docs/build-gn#gn)以设置`v8_enable_i18n_support = false`在编译之前。
