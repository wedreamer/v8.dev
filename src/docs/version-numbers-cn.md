***

## 标题：“V8 的版本编号方案”&#xA;描述：“本文档介绍了 V8 的版本编号方案。

V8 版本号的格式为`x.y.z.w`哪里：

*   `x.y`是铬里程碑除以 10（例如 M60 →`6.0`)
*   `z`每当有新的[断续器](https://chromium.googlesource.com/chromiumos/docs/+/main/glossary.md#acronyms)（通常每天几次）
*   `w`对于分支点之后的手动反向合并补丁

如果`w`是`0`，则从版本号中省略它。例如，v5.9.211（而不是“v5.9.211.0”）在回合并补丁后被提升到v5.9.211.1。

## 我应该使用哪个 V8 版本？

V8的嵌入器通常应使用*对应于Chrome中附带的V8次要版本的分支的负责人*.

### 查找与最新稳定版 Chrome 相对应的 V8 次要版本

要了解这是什么版本，

1.  转到(G)<https://omahaproxy.appspot.com/>
2.  在表格中查找最新的稳定版 Chrome 版本
3.  检查`v8_version`列（向右）在同一行上

示例：在撰写本文时，该网站指示`mac`/`stable`，Chrome 发布版本为 59.0.3071.86，对应于 V8 版本 5.9.211.31。

### 查找相应分支的负责人

V8 的与版本相关的分支不会出现在联机存储库中，网址为<https://chromium.googlesource.com/v8/v8.git>;而是仅显示标记。要查找该分支的负责人，请按以下形式转到 URL：

    https://chromium.googlesource.com/v8/v8.git/+/branch-heads/<minor-version>

示例：对于上面找到的 V8 次要版本 5.9，我们转到<https://chromium.googlesource.com/v8/v8.git/+/branch-heads/5.9>，找到标题为“版本 5.9.211.33”的提交。因此，在撰写本文时，嵌入器应使用的 V8 版本是**5.9.211.33**.

**谨慎：**你应该*不*只需找到与上述次要 V8 版本相对应的数值最大标记，因为有时这些标记不受支持，例如，在决定在哪里剪切次要版本之前，它们被标记。此类版本不会接收向后移植或类似内容。

示例：V8 标签`5.9.212`,`5.9.213`,`5.9.214`,`5.9.214.1`、...和`5.9.223`被遗弃，尽管在数量上大于**分支头**的 5.9.211.33.

### 检出相应分支的头部

如果您已经拥有源代码，则可以直接检查头部。如果您已使用`depot_tools`那么你应该能够做到

```bash
git branch --remotes | grep branch-heads/
```

以列出相关分支。您需要查看与您在上面找到的次要 V8 版本相对应的版本，然后使用它。您最终使用的标记是适合您作为嵌入器的 V8 版本。

如果您没有使用`depot_tools`编辑`.git/config`并将下面的行添加到`[remote "origin"]`部分：

    fetch = +refs/branch-heads/*:refs/remotes/branch-heads/*

示例：对于上面找到的 V8 次要版本 5.9，我们可以执行以下操作：

```bash
$ git checkout branch-heads/5.9
HEAD is now at 8c3db649d8... Version 5.9.211.33
```
