***

## 标题： “从源代码构建 V8”&#xA;描述： “本文档介绍如何从源代码构建 V8。

为了能够在适用于 x64 的 Windows/Linux/macOS 上从头开始构建 V8，请按照以下步骤操作。

## 获取 V8 源代码

按照我们指南中的说明进行操作[查看 V8 源代码](/docs/source-code).

## 安装构建依赖项

1.  对于 macOS：安装 Xcode 并接受其许可协议。（如果您单独安装了命令行工具，[首先删除它们](https://bugs.chromium.org/p/chromium/issues/detail?id=729990#c1).)

2.  确保您位于 V8 源目录中。如果您按照上一节中的每个步骤操作，那么您已经到达了正确的位置。

3.  下载所有构建依赖项：

    ```bash
    gclient sync
    ```

    对于 Google 用户 - 如果您在运行挂钩时看到“无法获取文件”或“需要登录”错误，请先尝试通过运行以下内容向 Google 存储进行身份验证：

    ```bash
    gsutil.py config
    ```

    使用您的 @google.com 帐户登录，然后输入`0`当被要求提供项目 ID 时。

4.  只有在 Linux 上才需要此步骤。安装其他构建依赖项：

    ```bash
    ./build/install-build-deps.sh
    ```

## V8栋

1.  确保您位于 V8 源目录下的`main`分支。

    ```bash
    cd /path/to/v8
    ```

2.  引入最新更改并安装任何新的构建依赖项：

    ```bash
    git pull && gclient sync
    ```

3.  编译源代码：

    ```bash
    tools/dev/gm.py x64.release
    ```

    或者，编译源代码并立即运行测试：

    ```bash
    tools/dev/gm.py x64.release.check
    ```

    有关`gm.py`帮助程序脚本及其触发的命令，请参阅[使用 GN 进行构建](/docs/build-gn).
