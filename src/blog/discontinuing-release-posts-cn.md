***

标题：“停止发布博客文章”
作者： '郭淑宇 （[@shu\_](https://twitter.com/\_shu))'
化身：

*   “舒玉国”
    日期： 2022-06-17
    标签：
*   释放
    描述： 'V8 停止发布博客文章，以支持Chrome发布时间表和功能博客文章。
    推文：“1537857497825824768”

***

从历史上看，V8 的每个新版本分支都有一篇博客文章。您可能已经注意到，自 v9.9 以来，还没有发布博客文章。从 v10.0 开始，我们将停止为每个新分支发布博客文章。但不要担心，您习惯于通过发布博客文章获得的所有信息仍然可用！请继续阅读，看看在哪里可以找到这些信息。

## 发布计划和当前版本

您是否正在阅读发布博客文章以确定 V8 的最新版本？

V8在Chrome的发布时间表上。有关 V8 的最新稳定版本，请查阅[Chrome 发布路线图](https://chromestatus.com/roadmap).

每四周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主分支出来的。这些分支处于测试阶段，并与[Chrome 发布路线图](https://chromestatus.com/roadmap).

要查找 Chrome 版本的特定 V8 分支，请执行以下操作：

1.  以Chrome版本除以10得到V8版本。例如，Chrome 102 是 V8 10.2。
2.  对于版本号 X.Y，可以在以下格式的 URL 中找到其分支：

<!---->

    https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/X.Y

例如，10.2 分支可以在<https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/10.2>.

有关版本号和分支的更多信息，请参阅[我们的详细文章](https://v8.dev/docs/version-numbers).

对于 V8 版本 X.Y，具有有效 V8 结账功能的开发人员可以使用`git checkout -b X.Y -t branch-heads/X.Y`以试验该版本中的新功能。

## 新的 JavaScript 或 WebAssembly 功能

您是否正在阅读发布博客文章，以了解哪些新的JavaScript或WebAssembly功能是在标志后面实现的，或者默认情况下是打开的？

请咨询[Chrome 发布路线图](https://chromestatus.com/roadmap)，其中列出了每个版本的新功能及其里程碑。

请注意，[单独的深入专题文章](/features)可以在 V8 中实现该功能之前或之后发布。

## 显著的性能改进

您是否正在阅读发布博客文章以了解显着的性能改进？

展望未来，我们将撰写独立的博客文章，以提高性能，我们希望指出，正如我们过去所做的那样，以进行改进，例如[火花塞](https://v8.dev/blog/sparkplug).

## 接口更改

您是否正在阅读发布博客文章以了解 API 更改？

要查看在早期版本 A.B 和更高版本 X.Y 之间修改 V8 API 的提交列表，请使用`git log branch-heads/A.B..branch-heads/X.Y include/v8\*.h`在活动的 V8 检出中。
