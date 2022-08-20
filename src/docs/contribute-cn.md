***

## 标题： “为V8做贡献”&#xA;描述： “本文档解释了如何为 V8 做贡献。

本页信息说明了如何为 V8 做贡献。在向我们发送贡献之前，请务必阅读整篇文章。

## 获取代码

看[查看 V8 源代码](/docs/source-code).

## 在您贡献之前

### 在 V8 的邮件列表上询问指导

在您开始进行更大的V8贡献之前，您应该首先通过以下方式与我们联系[V8 贡献者邮件列表](https://groups.google.com/group/v8-dev)因此，我们可以提供帮助，并可能为您提供指导。预先协调可以更容易地避免以后的挫折感。

### 签署共产国际

在我们使用您的代码之前，您必须签署[谷歌个人贡献者许可协议](https://cla.developers.google.com/about/google-individual)，您可以在线完成。这主要是因为您拥有更改的版权，即使您的贡献成为我们代码库的一部分，因此我们需要您的许可才能使用和分发您的代码。我们还需要确定其他各种事情，例如，如果您知道自己的代码侵犯了其他人的专利，您会告诉我们。在您提交代码以供审核并且成员批准之前，您不必执行此操作，但是在我们将您的代码放入我们的代码库之前，您必须这样做。

公司缴纳的会费受与上述协议不同的协议的约束，[软件授权和企业贡献者许可协议](https://cla.developers.google.com/about/google-corporate).

在线签名[这里](https://cla.developers.google.com/).

## 提交您的代码

V8 的源代码遵循[谷歌C++风格指南](https://google.github.io/styleguide/cppguide.html)因此，您应该熟悉这些准则。在提交代码之前，您必须通过我们所有的[测试](/docs/test)，并且必须成功运行预提交检查：

```bash
git cl presubmit
```

预提交脚本使用来自 Google 的 linter，[`cpplint.py`](https://raw.githubusercontent.com/google/styleguide/gh-pages/cpplint/cpplint.py).它是[`depot_tools`](https://dev.chromium.org/developers/how-tos/install-depot-tools)，并且它必须在您的`PATH`— 所以如果你有`depot_tools`在您的`PATH`，一切都应该正常工作。

### 上传到 V8 的代码审查工具

所有提交的内容，包括项目成员提交的材料，都需要审查。我们使用与Chromium项目相同的代码审查工具和流程。为了提交补丁，您需要获得[`depot_tools`](https://dev.chromium.org/developers/how-tos/install-depot-tools)并按照以下说明进行操作[请求审核](https://chromium.googlesource.com/chromium/src/+/master/docs/contributing.md)（使用 V8 工作区而不是 Chromium 工作区）。

### 注意破损或倒退

获得代码审查批准后，您可以使用提交队列登陆补丁。它运行一堆测试，如果所有测试都通过，则提交您的补丁。提交更改后，最好先观看[控制台](https://ci.chromium.org/p/v8/g/main/console)直到机器人在更改后变为绿色，因为控制台运行的测试比提交队列多一些。
