***

## 标题： “分类问题”&#xA;描述： “本文档介绍了如何处理 V8 的错误跟踪器中的问题。

本文档说明如何处理 中的问题[V8 的错误跟踪器](/bugs).

## 如何对问题进行会审

*   *V8 跟踪器*：将状态设置为`Untriaged`
*   *铬跟踪仪*：将状态设置为`Untriaged`并添加组件`Blink>JavaScript`

## 如何在铬跟踪器中分配 V8 问题

请将问题移至 V8 专业警长队列之一
以下类别：

*   记忆：`component:blink>javascript status=Untriaged label:Performance-Memory`
    *   将显示在[这](https://bugs.chromium.org/p/chromium/issues/list?can=2\&q=component%3Ablink%3Ejavascript+status%3DUntriaged+label%3APerformance-Memory+\&colspec=ID+Pri+M+Stars+ReleaseBlock+Cr+Status+Owner+Summary+OS+Modified\&x=m\&y=releaseblock\&cells=tiles)查询
*   稳定性：`status=available,untriaged component:Blink>JavaScript label:Stability -label:Clusterfuzz`
    *   将显示在[这](https://bugs.chromium.org/p/chromium/issues/list?can=2\&q=status%3Davailable%2Cuntriaged+component%3ABlink%3EJavaScript+label%3AStability+-label%3AClusterfuzz\&colspec=ID+Pri+M+Stars+ReleaseBlock+Component+Status+Owner+Summary+OS+Modified\&x=m\&y=releaseblock\&cells=ids)查询
    *   无需抄送，将由警长自动分类
*   性能：`status=untriaged component:Blink>JavaScript label:Performance`
    *   将显示在[这](https://bugs.chromium.org/p/chromium/issues/list?colspec=ID%20Pri%20M%20Stars%20ReleaseBlock%20Cr%20Status%20Owner%20Summary%20OS%20Modified\&x=m\&y=releaseblock\&cells=tiles\&q=component%3Ablink%3Ejavascript%20status%3DUntriaged%20label%3APerformance\&can=2)查询
    *   无需抄送，将由警长自动分类
*   集群模糊：将错误设置为以下状态：
    *   `label:ClusterFuzz component:Blink>JavaScript status:Untriaged`
    *   将显示在[这](https://bugs.chromium.org/p/chromium/issues/list?can=2\&q=label%3AClusterFuzz+component%3ABlink%3EJavaScript+status%3AUntriaged\&colspec=ID+Pri+M+Stars+ReleaseBlock+Component+Status+Owner+Summary+OS+Modified\&x=m\&y=releaseblock\&cells=ids)查询。
    *   无需抄送，将由警长自动分类
*   安全性：所有安全问题都由Chromium安全警长进行分类。请参阅[报告安全漏洞](/docs/security-bugs)了解更多信息。

如果您需要警长的注意，请查阅轮换信息。

使用组件`Blink>JavaScript`在所有问题上。

**请注意，这仅适用于在 Chromium 问题跟踪器中跟踪的问题。**
