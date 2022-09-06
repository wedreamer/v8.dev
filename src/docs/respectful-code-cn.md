***

## 标题： “尊重代码”&#xA;描述： “包容性是V8文化的核心，我们的价值观包括有尊严地对待彼此。因此，重要的是每个人都可以在不面对偏见和歧视的有害影响的情况下做出贡献。

包容性是 V8 文化的核心，我们的价值观包括有尊严地对待彼此。因此，重要的是每个人都可以在不面对偏见和歧视的有害影响的情况下做出贡献。但是，我们的代码库、UI 和文档中的术语可能会使这种歧视永久化。本文档列出了旨在解决代码和文档中不尊重术语的指南。

## 政策 { #policy }

应避免使用直接或间接的贬损、伤害或长期歧视的术语。

## 此策略的范围是什么？{ #scope }

贡献者在开发 V8 时会阅读的任何内容，包括：

*   变量的名称、类型、函数、文件、生成规则、二进制文件、导出的变量...
*   测试数据
*   系统输出和显示
*   文档（源文件内部和外部）
*   提交消息

## 原则 { #principles }

*   尊重：贬义语言不应该是描述事物如何运作的必要条件。
*   尊重文化敏感的语言：有些词可能具有重要的历史或政治含义。请注意这一点，并使用替代方案。

## 我如何知道特定术语是否可行？{ #questions }

应用上述原则。如果您有任何疑问，可以联系`v8-dev@googlegroups.com`.

## 有哪些应避免使用的术语示例？{ #examples }

此列表并不全面。它包含一些人们经常遇到的例子。

：：：表包装器
|学期|建议的替代方案|
|--------- |------------------------------------------------------------- |
|主|主服务器、控制器、控制器、主节点、主机|
|从属|副本、从属、辅助、从属、从属、设备、外围|
|白名单|白名单、例外名单、包含名单|
|黑名单|拒绝列表、阻止列表、排除列表|
|疯狂的|意想不到的、灾难性的、不连贯的|
|理智|预期、适当、合理、有效的|
|疯狂的|意想不到的、灾难性的、不连贯的|
|红线|优先线、限制、软限制|
:::

## 如果我与违反此政策的内容进行交互，该怎么办？{ #violations }

这种情况已经出现过几次，特别是对于代码实现规范。在这些情况下，与规范中的语言不同可能会干扰理解实现的能力。对于这些情况，我们建议以下之一，按优先级递减的顺序排列：

1.  如果使用替代术语不会干扰理解，请使用替代术语。
2.  如果做不到这一点，请不要将术语传播到执行接口的代码层之外。如有必要，请在 API 边界使用替代术语。