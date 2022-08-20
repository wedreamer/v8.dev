***

## 标题： 'V8的Linux`perf`集成'&#xA;描述： '本文档解释了如何使用 Linux 分析 V8 的 JITted 代码的性能`perf`工具。

V8 内置了对 Linux 的支持`perf`工具。它是由`--perf-prof`命令行选项。
V8 将执行过程中的性能数据写出到一个文件中，该文件可用于使用 Linux 分析 V8 的 JITted 代码（包括 JS 函数名称）的性能。`perf`工具。

## 要求

*   `linux-perf`版本 5 或更高版本（以前的版本不支持 jit）。（请参阅[结束](#build-perf))
*   构建 V8/Chrome`enable_profiling=true`以获得更好的符号化C++代码。

## V8栋

要使用 V8 与 Linux 性能的集成，您需要使用`enable_profiling = true`gn 标志：

```bash
echo 'enable_profiling = true' >> out/x64.release/args.gn
autoninja -C out/x64.release
```

## 分析`d8`跟[`linux-perf-d8.py`](https://source.chromium.org/search?q=linux-perf-d8.py)

建成后`d8`，你可以开始使用 linux 性能：

```bash
cd <path_to_your_v8_checkout>
echo '(function f() {
    var s = 0; for (var i = 0; i < 1000000000; i++) { s += i; } return s;
  })();' > test.js;
  
# Optional: create and output directory
mkdir profiling_out_dir;
tools/profiling/linux-perf-d8.py --perf-data-dir=profiling_out_dir \
    out/x64.release/d8 test.js;

# Fancy UI (`-flame` is googler-only, use `-web` as an alterntive):
pprof -flame profiling_out_dir/XXX_perf.data.jitted;
# Terminal-based tool:
perf report -i profiling_out_dir/XXX_perf.data.jitted;
```

检查`linux-perf-d8.py --help`了解更多详情。请注意，您可以使用所有`d8`标志之后`--`:

```bash
tools/profiling/linux-perf-d8.py --perf-data-dir=profiling_out_dir \
    -- out/x64.release/d8 --expose-gc --allow-natives-syntaxn test.js;
```

## 分析 Chrome 或 content_shell[linux-perf-chrome.py](https://source.chromium.org/search?q=linux-perf-chrome.py)

1.  您可以使用[linux-perf-chrome.py](https://source.chromium.org/search?q=linux-perf-chrome.py)脚本来分析镶边。确保添加[必需的铬 gn 标志](https://chromium.googlesource.com/chromium/src/+/master/docs/profiling.md#General-checkout-setup)以获得正确的C++符号。

2.  构建完成后，您可以使用C++和JS代码的完整符号来分析网站。

    ```bash
    mkdir perf_results;
    tools/profiling/linux-perf-chrome.py out/x64.release/chrome \
        --perf-data-dir=perf_results --timeout=30
    ```

3.  导航到您的网站，然后关闭浏览器（或等待`--timeout`完成）

4.  退出浏览器后`linux-perf.py`将对文件进行后处理，并显示一个列表，其中包含每个渲染器进程的结果文件：

        chrome_renderer_1583105_3.perf.data.jitted      19.79MiB
        chrome_renderer_1583105_2.perf.data.jitted       8.59MiB
        chrome_renderer_1583105_4.perf.data.jitted       0.18MiB
        chrome_renderer_1583105_1.perf.data.jitted       0.16MiB

## 探索 linux-perf 结果

最后，您可以使用Linux。`perf`工具浏览 d8 或镶边渲染器进程的配置文件：

```bash
perf report -i perf_results/XXX_perf.data.jitted
```

您还可以使用[普洛夫](https://github.com/google/pprof)生成更多可视化效果：

```bash
# Note: `-flame` is google-only, use `-web` as a public alterntive:
pprof -flame perf_results/XXX_perf.data.jitted;
```

## 低级 linux-perf 用法

### 使用 linux-perf 与`d8`径直

根据您的使用案例，您可能希望直接使用linux-perf`d8`.
这需要一个两步过程，首先`perf record`创建一个`perf.data`必须使用`perf inject`以注入 JS 符号。

```bash
perf record --call-graph=fp --clockid=mono --freq=max \
    --output=perf.data
    out/x64.release/d8 \
      --perf-prof --no-write-protect-code-memory \
      --interpreted-frames-native-stack \
    test.js;
perf inject --jit --input=perf.data --output=perf.data.jitted;
perf report --input=perf.data.jitted;
```

### V8 linux-perf Flags

[`--perf-prof`](https://source.chromium.org/search?q=FLAG_perf_prof)用于 V8 命令行，以在 JIT 代码中记录性能示例。

[`--nowrite-protect-code-memory`](https://source.chromium.org/search?q=FLAG_nowrite_protect_code_memory)要求禁用代码内存的写保护。这是必要的，因为`perf`当它看到与从代码页中删除写入位对应的事件时，将丢弃有关代码页的信息。下面是一个记录测试 JavaScript 文件中的示例：

[`--interpreted-frames-native-stack`](https://source.chromium.org/search?q=FLAG_interpreted_frames_native_stack)用于为解释函数创建不同的入口点（InterpreterEntryTrampoline的复制版本），以便可以通过以下方式区分它们`perf`仅基于地址。由于 InterpreterEntryTrampoline 必须复制，因此性能略有下降，内存回归。

### 直接将 linux-perf 与 chrome 结合使用

1.  您可以使用相同的 V8 标志来分析镶边本身。按照上面的说明获取正确的 V8 标志，并添加[必需的铬 gn 标志](https://chromium.googlesource.com/chromium/src/+/master/docs/profiling.md#General-checkout-setup)到您的镶边版本。

2.  构建完成后，您可以使用C++和JS代码的完整符号来分析网站。

    ```bash
    out/x64.release/chrome \
        --user-data-dir=`mktemp -d` \
        --no-sandbox --incognito --enable-benchmarking \
        --js-flags='--perf-prof --no-write-protect-code-memory --interpreted-frames-native-stack'
    ```

3.  启动chrome后，使用任务管理器找到渲染器进程ID，并使用它来开始分析：

    ```bash
    perf record -g -k mono -p $RENDERER_PID -o perf.data
    ```

4.  导航到您的网站，然后继续下一节，了解如何评估性能输出。

5.  执行完成后，合并从`perf`工具，其中包含 V8 为 JIT 代码输出的性能示例：

    ```bash
    perf inject --jit --input=perf.data --output=perf.data.jitted
    ```

6.  最后，您可以使用Linux。`perf` [工具探索](#Explore-linux-perf-results)

## 建`perf`

如果你有一个过时的linux内核，你可以在本地构建支持jit的linux-perf。

*   安装新的 Linux 内核，然后重新启动计算机：

    ```bash
     sudo apt-get install linux-generic-lts-wily;
    ```

*   安装依赖项：

    ```bash
    sudo apt-get install libdw-dev libunwind8-dev systemtap-sdt-dev libaudit-dev \
       libslang2-dev binutils-dev liblzma-dev;
    ```

*   下载包含最新内核源代码`perf`工具源：

    ```bash
    cd some/directory;
    git clone --depth 1 git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git;
    cd tip/tools/perf;
    make
    ```

在以下步骤中，调用`perf`如`some/director/tip/tools/perf/perf`.
