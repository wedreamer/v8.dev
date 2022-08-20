***

## 标题：“官方支持的配置”&#xA;描述：“本文档介绍了哪些构建配置由 V8 团队维护。

V8 支持跨操作系统、其版本、架构端口、构建标志等的多种不同的构建配置。

经验法则：如果我们支持它，我们就有一个机器人在我们的一个上运行。[持续集成控制台](https://ci.chromium.org/p/v8/g/main/console).

一些细微差别：

*   最重要的构建器上的破损将阻止代码提交。树木警长通常会恢复罪魁祸首。
*   破损大致相同[一组建筑商](https://chromium.googlesource.com/infra/infra/+/main/infra/services/lkgr_finder/config/v8\_cfg.pyl)阻止我们连续滚入铬。
*   一些架构端口是[外部处理](/docs/ports).
*   某些配置是[实验的](https://ci.chromium.org/p/v8/g/experiments/console).破损是允许的，将由配置的所有者处理。

如果您的配置存在问题，但未被上述机器人之一覆盖：

*   请随时提交解决您问题的 CL。该团队将为您提供代码审查支持。
*   您可以使用 v8-dev@googlegroups.com 来讨论问题。
*   如果您认为我们应该支持此配置（可能是我们的测试矩阵中的一个漏洞？），请在[V8 问题跟踪器](https://bugs.chromium.org/p/v8/issues/entry)并询问。

但是，我们没有带宽来支持每种可能的配置。
