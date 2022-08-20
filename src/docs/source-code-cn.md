***

## 标题：“查看 V8 源代码”&#xA;描述：“本文档介绍如何在本地查看 V8 源代码。

本文档介绍如何在本地检出 V8 源代码。如果您只想在线浏览源代码，请使用以下链接：

*   [浏览](https://chromium.googlesource.com/v8/v8/)
*   [浏览前沿](https://chromium.googlesource.com/v8/v8/+/master)
*   [变化](https://chromium.googlesource.com/v8/v8/+log/master)

## 使用 Git

V8 的 Git 存储库位于<https://chromium.googlesource.com/v8/v8.git>，在 GitHub 上有一个官方镜像：<https://github.com/v8/v8>.

不要只是`git clone`这些网址中的任何一个！如果你想从结账中构建V8，请按照下面的说明正确设置所有内容。

## 指示

1.  在 Linux 或 macOS 上，首先安装 Git，然后安装[`depot_tools`](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#\_setting_up).

    在 Windows 上，按照 Chromium 说明 （[对于谷歌员工](https://goto.google.com/building-chrome-win),[对于非谷歌用户](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md#Setting-up-Windows)） 来安装 Visual Studio、Windows 调试工具，以及`depot_tools`（在Windows上包括Git）。

2.  更新`depot_tools`通过在终端/shell 中执行以下命令。在 Windows 上，这必须在命令提示符 （`cmd.exe`），而不是PowerShell或其他。

        gclient

3.  为**推送访问**，您需要设置一个`.netrc`文件与你的 Git 密码：

    1.  转到(G)<https://chromium.googlesource.com/new-password>并使用您的提交者帐户登录（通常是`@chromium.org`帐户）。注意：创建新密码不会自动撤消以前创建的任何密码。请确保您使用与`git config user.email`.
    2.  看看包含 shell 命令的灰色大框。将这些线条粘贴到外壳中。

4.  现在，获取 V8 源代码，包括所有分支和依赖项：

    ```bash
    mkdir ~/v8
    cd ~/v8
    fetch v8
    cd v8
    ```

在那之后，你故意处于一个分离的头部状态。

（可选）您可以指定应如何跟踪新分支：

```bash
git config branch.autosetupmerge always
git config branch.autosetuprebase always
```

或者，您可以创建如下所示的新本地分支（推荐）：

```bash
git new-branch fix-bug-1234
```

## 保持最新状态

更新您当前的分行`git pull`.请注意，如果您不在分支上，`git pull`不起作用，您需要使用`git fetch`相反。

```bash
git pull
```

有时会更新 V8 的依赖项。您可以通过运行以下命令来同步这些内容：

```bash
gclient sync
```

## 发送代码以供审阅

```bash
git cl upload
```

## 犯

您可以使用代码审阅上的 CQ 复选框进行提交（首选）。另请参见[铬说明](https://chromium.googlesource.com/chromium/src/+/master/docs/infra/cq.md)用于 CQ 标志和故障排除。

如果您需要比默认值更多的 trybot，请在 Gerrit 上的提交消息中添加以下内容（例如，用于添加 nosnap 机器人）：

    CQ_INCLUDE_TRYBOTS=tryserver.v8:v8_linux_nosnap_rel

要手动登陆，请更新您的分行：

```bash
git pull --rebase origin
```

然后提交使用

```bash
git cl land
```

## 试用职位

本节仅对 V8 项目成员有用。

### 从代码审查创建试用作业

1.  将 CL 上传到 Gerrit。

    ```bash
    git cl upload
    ```

2.  通过向 try 机器人发送 try 作业来试用 CL，如下所示：

    ```bash
    git cl try
    ```

3.  等待尝试机器人生成，你会收到一封包含结果的电子邮件。您还可以在Gerrit上的补丁中检查尝试状态。

4.  如果应用修补程序失败，则需要重新调整修补程序的基值或指定要同步到的 V8 修订版：

```bash
git cl try --revision=1234
```

### 从本地分支创建试用作业

1.  将一些更改提交到本地存储库中的 git 分支。

2.  通过向 try 机器人发送 try 作业来尝试更改，如下所示：

    ```bash
    git cl try
    ```

3.  等待尝试机器人生成，你会收到一封包含结果的电子邮件。注意：目前某些副本存在问题。建议从 codereview 发送 try 作业。

### 有用的参数

修订参数告诉 try 机器人用于应用本地更改的代码库的哪个修订版。没有修订，[V8 的 LKGR 修订版](https://v8-status.appspot.com/lkgr)用作基础。

```bash
git cl try --revision=1234
```

若要避免在所有机器人上运行 try 作业，请使用`--bot`带有以逗号分隔的生成器名称列表的标志。例：

```bash
git cl try --bot=v8_mac_rel
```

### 查看试用服务器

```bash
git cl try-results
```

## 源代码分支

V8有几个不同的分支;如果您不确定要获取哪个版本，则很可能需要最新的稳定版本。看看我们的[发布流程](/docs/release-process)以获取有关所用不同分支的更多信息。

您可能希望关注 Chrome 在其稳定（或测试版）渠道上发布的 V8 版本，请参阅<https://omahaproxy.appspot.com/>.
