***

## 标题： '合并和修补'&#xA;描述： “本文档介绍如何将 V8 补丁合并到主分支。

如果您有一个补丁`master`分支（例如，一个重要的错误修复），需要合并到一个生产V8分支中，请继续阅读。

以下示例使用分支 2.4 版本的 V8。替代`2.4`以及您的版本号。阅读文档[V8的发布流程](/docs/release-process)和[V8 的版本编号](/docs/version-numbers)了解更多信息。

如果合并补丁，则 Chromium 或 V8 的问题跟踪器上的相关问题是必需的。这有助于跟踪合并。您可以使用[模板](https://code.google.com/p/v8/issues/entry?template=Merge%20request)以创建合并请求问题。

## 合并候选项的资格是什么？

*   该修补程序修复了*严重*错误（按重要性排序）：
    1.  安全漏洞
    2.  稳定性错误
    3.  正确性错误
    4.  性能错误
*   此修补程序不会更改 API。
*   该修补程序不会更改分支剪切之前存在的行为（除非行为更改修复了 Bug）。

更多信息可以在[相关铬页面](https://www.chromium.org/developers/the-zen-of-merge-requests).如有疑问，请发送电子邮件至<v8-dev@googlegroups.com>.

## 合并过程

Chromium和V8跟踪器中的合并过程由标签驱动，其形式为：

    Merge-[Status]-[Branch]

V8 目前重要的标签是：

1.  `Merge-Request-{Branch}`启动该过程，并表示应将此修复程序合并到`{Branch}`.`{Branch}`是 V8 分支的名称/编号，例如`7.2`对于M72。
2.  `Merge-Review-{Branch}`表示合并尚未获得批准`{Branch}`例如，因为缺少金丝雀保险。
3.  `Merge-Approved-{Branch}`表示 Chrome TPM 已签署合并协议。
4.  合并完成后，`Merge-Approved-{Branch}`标签替换为`Merge-Merged-{Branch}`.

## 如何检查提交是否已合并/还原/具有金丝雀覆盖范围

用`mergeinfo.py`获取连接到 的所有提交`$COMMIT_HASH`根据Git。

```bash
tools/release/mergeinfo.py $COMMIT_HASH
```

如果它告诉你`Is on Canary: No Canary coverage`您还不应该合并，因为修复程序尚未部署在 Canary 版本上。一个好的经验法则是在修复程序落地后至少等待 3 天，直到进行合并。

## 如何创建合并 CL

### 选项 1：使用[格里特](https://chromium-review.googlesource.com/)

请注意，仅当修补程序在发布分支上干净地应用时，此选项才有效。

1.  打开要向后合并的 CL。
2.  从扩展菜单中选择“樱桃采摘”（右上角有三个垂直点）。
3.  输入“引用/分支头/*X.X*“作为目标分支（替换*X.X*由适当的分支）。
4.  修改提交消息：
    1.  在标题前面加上前缀“合并：”。
    2.  从页脚中删除与原始 CL 对应的行（“更改 Id”、“已审阅”、“已审阅者”、“提交队列”、“Cr-Commit-Position”）。一定要保留“（从提交XXX中挑选的樱桃）”行，因为一些工具需要这样做才能将合并与原始CL相关联。
5.  如果发生合并冲突，请继续创建 CL。要解决冲突（如果有） - 使用gerrit UI，或者您可以使用菜单中的“下载补丁”命令（右上角的三个垂直点）轻松地在本地拉取补丁。
6.  发送以供审核。

### 选项 2：使用自动脚本

假设您正在将修订版 af3cf11 合并到分支 2.4（请指定完整的 git 哈希值 - 为简单起见，此处使用缩写）。

```bash
tools/release/merge_to_branch.py --branch 2.4 af3cf11
```

运行脚本`-h`以显示其帮助消息，其中包括更多选项（例如，您可以指定包含补丁的文件，也可以反转补丁，指定自定义提交消息或恢复之前取消的合并过程）。请注意，该脚本将使用 V8 的临时签出 - 它不会触及您的工作空间。您还可以一次合并多个修订版;只需将它们全部列出即可。

```bash
tools/release/merge_to_branch.py --branch 2.4 af3cf11 cf33f1b sf3cf09
```

### 着陆后：观察[分支瀑布](https://ci.chromium.org/p/v8)

如果其中一个构建器在处理您的补丁后不是绿色的，请立即恢复合并。机器人 （`AutoTagBot`） 在等待 10 分钟后负责正确的版本控制。

## 修补金丝雀/开发上使用的版本

如果您需要修补金丝雀/开发版本（这应该不经常发生），请按照以下说明进行操作。谷歌人：请查看[内部站点](http://g3doc/company/teams/v8/patching_a_version)在创建 CL 之前。

### 步骤 1：合并到滚动分支

使用的示例版本是`5.7.433`.

```bash
tools/release/roll_merge.py --branch 5.7.433 af3cf11
```

### 步骤2：使铬识别固定

使用的铬分支示例是`2978`:

```bash
git checkout chromium/2978
git merge 5.7.433.1
git push
```

### 第3步：结束

Chrome/Chromium应该在自动构建时拾取更改。

## 常见问题

### 我在合并过程中收到与标记相关的错误。我该怎么办？

当两个人同时合并时，合并脚本中可能会发生争用情况。如果是这种情况，请联系<machenbach@chromium.org>和<hablich@chromium.org>.

### 是否有 TL;DR？

1.  [在问题跟踪器上创建问题](https://bugs.chromium.org/p/v8/issues/entry?template=Merge%20request).
2.  检查修复程序的状态`tools/release/mergeinfo.py`
3.  加`Merge-Request-{Branch}`到问题。
4.  等到有人添加`Merge-Approved-{Branch}`.
5.  [合并](#step-1-run-the-script).
