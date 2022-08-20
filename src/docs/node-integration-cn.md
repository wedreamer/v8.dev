***

## 标题： “如果你的 CL 破坏了 Node.js集成构建，该怎么办”&#xA;描述：“本文档说明如果您的 CL 破坏了 Node.js 集成构建，该怎么办。

[节点.js](https://github.com/nodejs/node)使用 V8 稳定版或测试版。为了进行额外的集成，V8 团队使用 V8 的[主分支](https://chromium.googlesource.com/v8/v8/+/refs/heads/main)，即从今天开始使用V8版本。我们提供集成机器人[Linux](https://ci.chromium.org/p/node-ci/builders/ci/Node-CI%20Linux64)而[窗户](https://ci.chromium.org/p/node-ci/builders/ci/Node-CI%20Win64)和[苹果电脑](https://ci.chromium.org/p/node-ci/builders/ci/Node-CI%20Mac64)正在制作中。

如果[`node_ci_linux64_rel`](https://ci.chromium.org/p/node-ci/builders/try/node_ci_linux64\_rel)机器人在 V8 提交队列上失败，您的 CL 存在合法问题（修复它）或[节点](https://github.com/v8/node/)必须修改。如果节点测试失败，请在日志文件中搜索“不正常”。**本文档介绍如何在本地重现问题以及如何对[V8 的节点分叉](https://github.com/v8/node/)如果 V8 CL 导致构建失败。**

## 源

按照[指示](https://chromium.googlesource.com/v8/node-ci)在 node-ci 存储库中签出源代码。

## 测试对 V8 的更改

V8 被设置为 node-ci 的 DEPS 依赖项。您可能希望将更改应用于 V8 以进行测试或重现故障。为此，请将主 V8 检出添加为远程：

```bash
cd v8
git remote add v8 <your-v8-dir>/.git
git fetch v8
git checkout v8/<your-branch>
cd ..
```

请记住在编译之前运行 gclient hooks。

```bash
gclient runhooks
JOBS=`nproc` make test
```

## 对节点进行更改.js

Node.js 也设置为`DEPS`node-ci 的依赖关系。您可能希望将更改应用于 Node.js以修复 V8 更改可能导致的中断。V8 针对[节点的分叉.js](https://github.com/v8/node).您需要一个 GitHub 帐户才能对该分叉进行更改。

### 获取节点源

叉[V8 的 Node.js GitHub 上的存储库](https://github.com/v8/node/)（单击分叉按钮），除非您之前已经这样做过。

将你的分叉和V8的分叉作为遥控器添加到现有的结账中：

```bash
cd node
git remote add v8 http://github.com/v8/node
git remote add <your-user-name> git@github.com:<your-user-name>/node.git
git fetch v8
git checkout node-ci-<sync-date>
export BRANCH_NAME=`date +"%Y-%m-%d"`_fix_name
git checkout -b $BRANCH_NAME
```

对 Node 进行更改.js签出，然后提交它们。然后将更改推送到 GitHub，并针对分支创建拉取请求`node-ci-<sync-date>`.

：：：备注
**注意：** `<sync-date>`是我们与上游 Node 同步的日期.js。选择最晚的日期。
:::

如果您的更改最终需要上游，请添加标记`upstream`与支持的 V8 版本，例如`[upstream:v8.4.377]`.如果您的更改是临时的（例如禁用测试），请添加标记`temp`以及应放弃更改的 V8 版本，例如：`[temp:v8.4.377]`.

```bash
git push <your-user-name> $BRANCH_NAME
```

一旦拉取请求合并到 V8 的 Node.js 分支，您需要更新 node-ci 的`DEPS`文件，然后创建 CL。

```bash
git checkout -b update-deps
gclient setdep --var=node_revision=`(cd node && git rev-parse $BRANCH_NAME)`
git add DEPS
git commit -m 'Update Node'
git cl upload
```
