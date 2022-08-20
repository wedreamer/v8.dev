***

标题： “导入断言”
作者： '丹·克拉克 （[@dandclark1](https://twitter.com/dandclark1)），导入断言的断言的断言
化身：

*   “丹-克拉克”
    日期： 2021-06-15
    标签：
*   ECMAScript
    描述：“导入断言允许模块导入语句在模块说明符旁边包含其他信息”
    推特： ''

***

新[导入断言](https://github.com/tc39/proposal-import-assertions)功能允许模块导入语句在模块说明符旁边包含其他信息。该功能的初始用途是启用 JSON 文档作为导入[JSON 模块](https://github.com/tc39/proposal-json-modules):

```json
// foo.json
{ "answer": 42 }
```

```javascript
// main.mjs
import json from './foo.json' assert { type: 'json' };
console.log(json.answer); // 42
```

## 背景：JSON 模块和 MIME 类型

一个自然而然的问题是，为什么JSON模块不能像这样简单地导入：

```javascript
import json from './foo.json';
```

Web 平台在执行模块资源之前检查其有效性，从理论上讲，此 MIME 类型也可用于确定是将资源视为 JSON 还是 JavaScript 模块。

但是，有一个[安全问题](https://github.com/w3c/webcomponents/issues/839)仅依靠MIME类型。

模块可以跨源导入，开发人员可以从第三方源导入 JSON 模块。只要正确清理了JSON，他们可能会认为这是基本的，即使来自不受信任的第三方，因为导入JSON不会执行脚本。

但是，第三方脚本实际上可以在这种情况下执行，因为第三方服务器可能会意外地使用 JavaScript MIME 类型和恶意 JavaScript 有效负载进行回复，从而在导入程序的域中运行代码。

```javascript
// Executes JS if evil.com responds with a
// JavaScript MIME type (e.g. `text/javascript`)!
import data from 'https://evil.com/data.json';
```

文件扩展名不能用于进行模块类型确定，因为它们[不是 Web 上内容类型的可靠指标](https://github.com/tc39/proposal-import-assertions/blob/master/content-type-vs-file-extension.md).因此，我们使用导入断言来指示预期的模块类型，并防止这种权限升级陷阱。

当开发人员想要导入 JSON 模块时，他们必须使用导入断言来指定它应该是 JSON。如果从网络接收的 MIME 类型与预期类型不匹配，则导入将失败：

```javascript
// Fails if evil.com responds with a non-JSON MIME type.
import data from 'https://evil.com/data.json' assert { type: 'json' };
```

## 动态`import()`

导入断言也可以传递给[动态`import()`](https://v8.dev/features/dynamic-import#dynamic)使用新的第二个参数：

```json
// foo.json
{ "answer": 42 }
```

```javascript
// main.mjs
const jsonModule = await import('./foo.json', {
  assert: { type: 'json' }
});
console.log(jsonModule.default.answer); // 42
```

JSON 内容是模块的默认导出，因此通过`default`从 返回的对象上的属性`import()`.

## 结论

目前，导入断言的唯一指定用途是指定模块类型。但是，该功能被设计为允许任意键/值断言对，因此，如果以其他方式限制模块导入变得有用，将来可能会添加其他用途。

同时，具有新的导入断言语法的 JSON 模块在 Chromium 91 中默认可用。[CSS 模块脚本](https://chromestatus.com/feature/5948572598009856)也即将推出，使用相同的模块类型断言语法。

## 导入断言支持 { #support }

<功能支持 chrome=“91 https://chromestatus.com/feature/5765269513306112” firefox=“no” safari=“no” nodejs=“no” babel=“yes https://github.com/babel/babel/pull/12139”></feature-support>
