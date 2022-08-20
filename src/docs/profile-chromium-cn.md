***

## 标题： “用V8分析铬”&#xA;描述： “本文档介绍如何将 V8 的 CPU 和堆分析器与 Chromium 配合使用。

[V8 的 CPU 和堆分析器](/docs/profile)从V8的外壳中使用是微不足道的，但是如何将它们与Chromium一起使用可能会令人困惑。此页面应该可以帮助您。

## 为什么将 V8 的轮廓仪与 Chromium 配合使用与在 V8 外壳上使用不同？

铬是一个复杂的应用，不像V8外壳。以下是影响探查器使用的铬功能列表：

*   每个渲染器都是一个单独的进程（好吧，实际上不是每个，但让我们省略这个细节），所以它们不能共享相同的日志文件;
*   围绕渲染器进程构建的沙箱可防止其写入磁盘;
*   开发人员工具根据自己的目的配置分析器;
*   V8 的日志记录代码包含一些优化，以简化日志记录状态检查。

## 如何运行Chromium来获取CPU配置文件？

以下是如何运行Chromium以便从该过程开始时获取CPU配置文件：

```bash
./Chromium --no-sandbox --user-data-dir=`mktemp -d` --incognito --js-flags='--prof'
```

请注意，您不会在开发人员工具中看到配置文件，因为所有数据都记录到文件中，而不是记录到开发人员工具中。

### 标志说明

`--no-sandbox`关闭渲染器沙盒，以便 chrome 可以写入日志文件。

`--user-data-dir`用于创建新的配置文件，使用它来避免缓存和已安装扩展的潜在副作用（可选）。

`--incognito`用于进一步防止结果污染（可选）。

`--js-flags`包含传递给 V8 的标志：

*   `--logfile=%t.log`指定日志文件的名称模式。`%t`以毫秒为单位扩展到当前时间，因此每个进程都有自己的日志文件。如果需要，可以使用前缀和后缀，如下所示：`prefix-%t-suffix.log`.默认情况下，每个隔离项都会获取一个单独的日志文件。
*   `--prof`告诉 V8 将统计分析信息写入日志文件。

## 人造人

Android上的Chrome有许多独特的观点，使其分析起来更加复杂。

*   命令行必须通过以下方式编写`adb`在设备上启动 Chrome 之前。因此，命令行中的引号有时会丢失，最好将参数分开`--js-flags`使用逗号，而不是尝试使用空格和引号。
*   日志文件的路径必须指定为指向 Android 文件系统上可写位置的绝对路径。
*   Android上用于渲染器进程的沙盒意味着即使使用`--no-sandbox`，则渲染器进程仍无法写入文件系统上的文件，因此`--single-process`需要传递才能在与浏览器进程相同的进程中运行呈现器。
*   这`.so`嵌入在Chrome的APK中，这意味着符号化需要从APK内存地址转换为未剥离的`.so`文件中。

以下命令可在 Android 上启用性能分析：

```bash
./build/android/adb_chrome_public_command_line --no-sandbox --single-process --js-flags='--logfile=/storage/emulated/0/Download/%t.log,--prof'
<Close and relaunch Chome on the Android device>
adb pull /storage/emulated/0/Download/<logfile>
./src/v8/tools/linux-tick-processor --apk-embedded-library=out/Release/lib.unstripped/libchrome.so --preprocess <logfile>
```

## 笔记

在“窗口”下，请务必打开`.MAP`的文件创建`chrome.dll`，但不适用于`chrome.exe`.
