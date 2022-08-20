***

## 标题： “发布流程”&#xA;描述：“本文档介绍了 V8 的发布过程。

V8 发布流程与[铬的](https://www.chromium.org/getting-involved/dev-channel).V8团队正在使用所有四个Chrome发布渠道向用户推送新版本。

如果您想查找Chrome版本中的V8版本，可以检查[奥马哈普罗西](https://omahaproxy.appspot.com/).对于每个Chrome版本，都会在V8存储库中创建一个单独的分支，以使回溯更容易，例如[铬 94.0.4606.61](https://chromium.googlesource.com/v8/v8.git/+/chromium/4606).

## 金丝雀发行

每天都有新的金丝雀版本被推送给用户[Chrome的金丝雀频道](https://www.google.com/chrome/browser/canary.html?platform=win64).通常，可交付成果是最新，足够稳定的版本[主要](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/main).

金丝雀的分支通常如下所示：

    remotes/origin/9.4.146

## 开发版本

每周都会通过以下方式向用户推送新的 Dev 版本[Chrome 的 Dev 频道](https://www.google.com/chrome/browser/desktop/index.html?extra=devchannel\&platform=win64).通常，可交付成果包括金丝雀频道上足够稳定的最新V8版本。

Dev 的分支通常如下所示：

    remotes/origin/9.4.146

## 测试版

大约每4周就会创建一个新的主要分支，例如[适用于铬 94](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.4).这与[Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html?platform=win64).Chrome Beta被固定在V8分支的头部。大约4周后，分支被提升为稳定。

更改仅精选到分支上，以稳定版本。

Beta 版的分支通常如下所示

    remotes/branch-heads/9.4

它们基于金丝雀分支。

## 稳定版本

大约每4周就会有一个新的主要稳定版本完成。不会创建特殊分支，因为最新的 Beta 分支只是简单地提升到稳定分支。此版本通过以下方式推送给用户[Chrome 的稳定渠道](https://www.google.com/chrome/browser/desktop/index.html?platform=win64).

稳定版本的分支通常如下所示：

    remotes/branch-heads/9.4

它们是升级（重用）的 Beta 分支。

## 我应该在应用程序中嵌入哪个版本？

Chrome 的稳定频道使用的同一分支的尖端。

我们经常将重要的错误修复反向合并到一个稳定的分支中，所以如果你关心稳定性、安全性和正确性，你也应该包括这些更新——这就是为什么我们建议使用“分支的尖端”，而不是一个确切的版本。

一旦新分支晋升为稳定分支，我们就会停止维护以前的稳定分支。这种情况每六周发生一次，因此您应该准备好至少经常更新。

示例：如果当前的稳定版 Chrome 版本是[94.0.4606.61](https://omahaproxy.appspot.com)，与 V8 v9.4.146.17。所以你应该嵌入[分支头/9.4](https://chromium.googlesource.com/v8/v8.git/+/branch-heads/9.4).当Chrome 95在稳定频道上发布时，您应该更新到branch-heads/9.5。

**相关：** [我应该使用哪个 V8 版本？](/docs/version-numbers#which-v8-version-should-i-use%3F)
