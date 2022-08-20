***

标题： “关于 Node 中的哈希泛洪漏洞.js...”。
作者： '杨果 （[@hashseed](https://twitter.com/hashseed))'
化身：

*   “阳国”
    日期： 2017-08-11 13：33：37
    标签：
*   安全
    描述： 'Node.js遭受哈希泛洪漏洞。这篇文章提供了一些背景知识，并解释了V8中的解决方案。

***

今年7月初，Node.js发布了一个[安全更新](https://nodejs.org/en/blog/vulnerability/july-2017-security-releases/)用于所有当前维护的分支，以解决哈希泛洪漏洞。这种中间修复是以启动性能大幅下降为代价的。同时，V8 还实现了一个解决方案，避免了性能损失。

在这篇文章中，我们想提供有关漏洞和最终解决方案的一些背景和历史。

## 哈希泛洪攻击

哈希表是计算机科学中最重要的数据结构之一。它们在 V8 中被广泛使用，例如用于存储对象的属性。平均而言，插入新条目在[O（1）](https://en.wikipedia.org/wiki/Big_O_notation).但是，哈希冲突可能导致最坏的 O（n） 情况。这意味着插入 n 个条目最多可能需要 O（n²）。

在节点.js，[HTTP 标头](https://nodejs.org/api/http.html#http_response_getheaders)表示为 JavaScript 对象。标头名称和值对存储为对象属性。通过巧妙准备的HTTP请求，攻击者可以执行拒绝服务攻击。Node.js进程将变得无响应，忙于最坏情况的哈希表插入。

这种攻击早在[2011年12月](https://events.ccc.de/congress/2011/Fahrplan/events/4680.en.html)，并被证明会影响广泛的编程语言。为什么V8和Node.js花了这么长时间才最终解决这个问题？

事实上，在披露后不久，V8工程师就与Node.js社区合作开发了一个[缓解](https://github.com/v8/v8/commit/81a0271004833249b4fe58f7d64ae07e79cffe40).从 Node.js v0.11.8 开始，此问题已得到解决。该修复引入了所谓的*哈希种子值*.哈希种子在启动时随机选择，用于为特定 V8 实例中的每个哈希值设定种子。如果不了解哈希种子，攻击者很难达到最坏的情况，更不用说提出针对所有Node.js实例的攻击了。

这是[犯](https://github.com/v8/v8/commit/81a0271004833249b4fe58f7d64ae07e79cffe40)修复消息：

> 此版本仅解决了那些自己编译 V8 或不使用快照的用户的问题。基于快照的预编译 V8 仍将具有可预测的字符串哈希代码。

此版本仅解决了那些自己编译 V8 或不使用快照的用户的问题。基于快照的预编译 V8 仍将具有可预测的字符串哈希代码。

## 启动快照

启动快照是 V8 中的一种机制，可显著加快引擎启动速度并创建新上下文（即通过[虚拟机模块](https://nodejs.org/api/vm.html)在节点中.js）。V8 不是从头开始设置初始对象和内部数据结构，而是从现有快照进行反序列化。具有快照的 V8 最新版本在不到 3 毫秒的时间内启动，并且需要几分之一毫秒才能创建新上下文。如果没有快照，启动时间超过 200 毫秒，新上下文需要 10 毫秒以上。这是两个数量级的差异。

我们介绍了任何 V8 嵌入器如何利用[上一篇文章](/blog/custom-startup-snapshots).

预构建的快照包含哈希表和其他基于哈希值的数据结构。从快照初始化后，无法再在不损坏这些数据结构的情况下更改哈希种子。捆绑快照的 Node.js 版本具有固定的哈希种子，使缓解措施无效。

这就是提交消息中的显式警告的内容。

## 几乎固定，但不完全

快进到2015年，一个节点.js[问题](https://github.com/nodejs/node/issues/1631)报告创建新上下文的性能有所下降。不出所料，这是因为作为缓解措施的一部分，启动快照已被禁用。但到那时，并不是每个参与讨论的人都意识到[原因](https://github.com/nodejs/node/issues/528#issuecomment-71009086).

如本文所述[发布](/blog/math-random)，V8 使用伪随机数生成器生成 Math.random 结果。每个 V8 上下文都有自己的随机数生成状态副本。这是为了防止 Math.random 结果在上下文中可预测。

随机数生成器状态是在创建上下文后立即从外部源设定种子的。上下文是从头开始创建，还是从快照反序列化，都无关紧要。

不知何故，随机数生成器状态已经[困惑](https://github.com/nodejs/node/issues/1631#issuecomment-100044148)使用哈希种子。因此，自那以后，预构建的快照开始成为官方发布的一部分。[io.js v2.0.2](https://github.com/nodejs/node/pull/1679).

## 第二次尝试

直到2017年5月，在V8之间的一些内部讨论中，[谷歌的零项目](https://googleprojectzero.blogspot.com/)和谷歌的Cloud Platform，当我们意识到Node.js仍然容易受到哈希泛滥攻击。

最初的回应来自我们的同事[阿里](https://twitter.com/ofrobots)和[迈尔斯](https://twitter.com/MylesBorins)从背后的团队[Google Cloud Platform 的 Node.js产品](https://cloud.google.com/nodejs/).他们与Node.js社区合作[禁用启动快照](https://github.com/nodejs/node/commit/eff636d8eb7b009c40fb053802c169ba1417293d)默认情况下，再次。这一次，他们还添加了一个[测试用例](https://github.com/nodejs/node/commit/9fedc1f09648ff7cebed65883966f5647686a38a).

但我们不想就此打住。禁用启动快照具有[重要](https://github.com/nodejs/node/issues/14229)性能影响。多年来，我们增加了许多新的[语言](/blog/high-performance-es2015)  [特征](/blog/webassembly-browser-preview)和[复杂](/blog/launching-ignition-and-turbofan)  [优化](/blog/speeding-up-regular-expressions)到 V8。其中一些新增功能使得从头开始变得更加昂贵。在安全发布之后，我们立即开始研究一个长期解决方案。目标是能够[重新启用启动快照](https://github.com/nodejs/node/issues/14171)而不会变得容易受到哈希泛滥的影响。

从[建议的解决方案](https://docs.google.com/document/d/1br7T3jk5JAJSYaT8eZdQlqrPTDRClheGpRU1-BpY1ss/edit)，我们选择并实施了最务实的一个。从快照反序列化后，我们将选择一个新的哈希种子。然后重新哈希处理受影响的数据结构以确保一致性。

事实证明，在普通的启动快照中，很少有数据结构实际上受到影响。令我们高兴的是，[重新哈希表](https://github.com/v8/v8/commit/0e8e0030775518b69eb8522823ea3754e6bddc69)在此期间，在 V8 中变得很容易。这增加的开销是微不足道的。

重新启用启动快照的修补程序已[合并](https://github.com/nodejs/node/commit/2ae2874ae7dfec2c55b5d390d25b6eed9932f78d) [到](https://github.com/nodejs/node/commit/14e4254f68f71a6afaf3ebe16794172b08e68d7b)Node.js. 它是最近的 Node.js v8.3.0 的一部分[释放](https://medium.com/the-node-js-collection/node-js-8-3-0-is-now-available-shipping-with-the-ignition-turbofan-execution-pipeline-aa5875ad3367).
