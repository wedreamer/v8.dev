***

标题： '模块命名空间导出'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-12-18
    标签：
*   ECMAScript
*   ES2020
    描述：“JavaScript 模块现在支持新的语法来重新导出命名空间中的所有属性。

***

在[JavaScript 模块](/features/modules)，则已经可以使用以下语法：

```js
import * as utils from './utils.mjs';
```

但是，没有对称`export`语法存在...[直到现在](https://github.com/tc39/proposal-export-ns-from):

```js
export * as utils from './utils.mjs';
```

这等效于以下内容：

```js
import * as utils from './utils.mjs';
export { utils };
```
