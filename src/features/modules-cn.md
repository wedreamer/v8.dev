***

标题： 'JavaScript modules'
作者： '阿迪·奥斯曼尼 （[@addyosmani](https://twitter.com/addyosmani)）和马蒂亚斯·拜恩斯（[@mathias](https://twitter.com/mathias))'
化身：

*   'addy-osmani'
*   'mathias-bynens'
    日期： 2018-06-18
    标签：
    *   ECMAScript
    *   ES2015
        描述： “本文解释了如何使用JavaScript模块，如何负责任地部署它们，以及Chrome团队如何努力使模块在未来变得更好。
        推文：“1008725884575109120”

***

JavaScript 模块现在是[在所有主流浏览器中均受支持](https://caniuse.com/#feat=es6-module)!

<feature-support chrome="61"
              firefox="60"
              safari="11"
              nodejs="13.2.0 https://nodejs.org/en/blog/release/v13.2.0/#notable-changes"
              babel="yes"></feature-support>

本文将介绍如何使用 JS 模块，如何负责任地部署它们，以及 Chrome 团队如何努力使模块在未来变得更好。

## 什么是 JS 模块？{ #intro }

JS 模块（也称为“ES 模块”或“ECMAScript 模块”）是一个主要的新功能，或者更确切地说是新功能的集合。您可能过去使用过用户域 JavaScript 模块系统。也许你用过[CommonJS，如 Node.js](https://nodejs.org/docs/latest-v10.x/api/modules.html)，或者也许[阿德姆](https://github.com/amdjs/amdjs-api/blob/master/AMD.md)，或者其他什么。所有这些模块系统都有一个共同点：它们允许您导入和导出内容。

JavaScript现在已经实现了标准化的语法。在模块中，您可以使用`export`关键字导出几乎任何东西。您可以导出`const`一个`function`，或任何其他变量绑定或声明。只需在变量语句或声明前面加上前缀`export`你都准备好了：

```js
// 📁 lib.mjs
export const repeat = (string) => `${string} ${string}`;
export function shout(string) {
  return `${string.toUpperCase()}!`;
}
```

然后，您可以使用`import`关键字，以从另一个模块导入模块。在这里，我们正在导入`repeat`和`shout`功能从`lib`模块，并将其用于我们的`main`模块：

```js
// 📁 main.mjs
import {repeat, shout} from './lib.mjs';
repeat('hello');
// → 'hello hello'
shout('Modules in action');
// → 'MODULES IN ACTION!'
```

您还可以导出*违约*模块中的值：

```js
// 📁 lib.mjs
export default function(string) {
  return `${string.toUpperCase()}!`;
}
```

这样`default`可以使用任何名称导入导出：

```js
// 📁 main.mjs
import shout from './lib.mjs';
//     ^^^^^
```

模块与经典脚本略有不同：

*   模块具有[严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)默认情况下启用。

*   HTML 样式的注释语法在模块中不受支持，尽管它在经典脚本中有效。

    ```js
    // Don’t use HTML-style comment syntax in JavaScript!
    const x = 42; <!-- TODO: Rename x to y.
    // Use a regular single-line comment instead:
    const x = 42; // TODO: Rename x to y.
    ```

*   模块具有词法顶级范围。这意味着例如，运行`var foo = 42;`在模块中*不*创建一个名为 的全局变量`foo`，可通过以下方式访问`window.foo`在浏览器中，尽管在经典脚本中就是这种情况。

*   同样，`this`在模块内不是指全局`this`，而是`undefined`.（用途[`globalThis`](/features/globalthis)如果您需要访问全球`this`.)

*   新的静态`import`和`export`语法仅在模块中可用 - 它在经典脚本中不起作用。

*   [顶级`await`](/features/top-level-await)在模块中可用，但在经典脚本中不可用。相关，`await`不能用作模块中任何位置的变量名，尽管经典脚本中的变量*能*被命名`await`异步函数之外。

由于这些差异，*相同的JavaScript代码在被视为模块与经典脚本时的行为可能不同*.因此，JavaScript 运行时需要知道哪些脚本是模块。

## 在浏览器中使用 JS 模块 { #browser }

在网络上，您可以告诉浏览器处理`<script>`元素作为模块，方法是将`type`属性为`module`.

```html
<script type="module" src="main.mjs"></script>
<script nomodule src="fallback.js"></script>
```

了解的浏览器`type="module"`忽略脚本`nomodule`属性。这意味着您可以将基于模块的有效负载提供给支持模块的浏览器，同时提供对其他浏览器的回退。做出这种区分的能力是惊人的，即使只是为了性能！想想看：只有现代浏览器支持模块。如果浏览器理解您的模块代码，它还支持[模块之前的功能](https://codepen.io/samthor/pen/MmvdOM)，例如箭头函数或`async`-`await`.您不必再在模块包中转译这些功能了！您可以[为现代浏览器提供更小且基本上未传输的基于模块的有效负载](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/).只有旧版浏览器才能获得`nomodule`有效载荷。

因为[默认情况下，模块被延迟](#defer)，您可能需要加载`nomodule`以延迟方式编写脚本：

```html
<script type="module" src="main.mjs"></script>
<script nomodule defer src="fallback.js"></script>
```

### 模块和经典脚本之间特定于浏览器的差异 { #module-vs-script }

如您现在所知，模块与经典脚本不同。除了我们上面概述的与平台无关的差异之外，还有一些特定于浏览器的差异。

例如，模块仅计算一次，而经典脚本则在将经典脚本添加到 DOM 的次数之后进行评估。

```html
<script src="classic.js"></script>
<script src="classic.js"></script>
<!-- classic.js executes multiple times. -->

<script type="module" src="module.mjs"></script>
<script type="module" src="module.mjs"></script>
<script type="module">import './module.mjs';</script>
<!-- module.mjs executes only once. -->
```

此外，模块脚本及其依赖项是使用 CORS 获取的。这意味着任何跨域模块脚本都必须使用正确的标头，例如`Access-Control-Allow-Origin: *`.对于经典脚本，情况并非如此。

另一个区别涉及`async`属性，这会导致脚本下载而不阻塞 HTML 解析器（如`defer`），除了它还会尽快执行脚本，没有保证顺序，也不需要等待 HTML 解析完成。这`async`属性不适用于内联经典脚本，但适用于内联脚本`<script type="module">`.

### 关于文件扩展名的说明 { #mjs }

您可能已经注意到我们正在使用`.mjs`模块的文件扩展名。在 Web 上，文件扩展名并不重要，只要文件与[JavaScript MIME 类型`text/javascript`](https://html.spec.whatwg.org/multipage/scripting.html#scriptingLanguages:javascript-mime-type).浏览器知道它是一个模块，因为`type`属性。

不过，我们建议使用`.mjs`模块的扩展，原因有二：

1.  在开发过程中，`.mjs`扩展使您和其他查看您的项目的人清楚地知道该文件是一个模块，而不是一个经典脚本。（并不总是能够通过查看代码来判断。如前所述，模块的处理方式与经典脚本不同，因此差异非常重要！
2.  它确保您的文件被运行时解析为模块，例如[节点.js](https://nodejs.org/api/esm.html#esm_enabling)和[`d8`](/docs/d8)，并构建工具，例如[巴别塔](https://babeljs.io/docs/en/options#sourcetype).虽然这些环境和工具都有专有的配置方式来解释具有其他扩展名的文件作为模块，但`.mjs`扩展是确保将文件视为模块的交叉兼容方式。

：：：备注
**注意：**部署`.mjs`在 Web 上，需要将 Web 服务器配置为使用适当的扩展名提供具有此扩展名的文件`Content-Type: text/javascript`标头，如上所述。此外，您可能希望将编辑器配置为`.mjs`文件作为`.js`文件以获取语法突出显示。默认情况下，大多数现代编辑器已经执行此操作。
:::

### 模块说明符 { #specifiers }

什么时候`import`对于模块，指定模块位置的字符串称为“模块说明符”或“导入说明符”。在我们前面的示例中，模块说明符是`'./lib.mjs'`:

```js
import {shout} from './lib.mjs';
//                  ^^^^^^^^^^^
```

某些限制适用于浏览器中的模块说明符。目前不支持所谓的“裸”模块说明符。此限制是[指定](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)以便将来，浏览器可以允许自定义模块加载器为裸模块说明符赋予特殊含义，如下所示：

```js
// Not supported (yet):
import {shout} from 'jquery';
import {shout} from 'lib.mjs';
import {shout} from 'modules/lib.mjs';
```

另一方面，以下示例均受支持：

```js
// Supported:
import {shout} from './lib.mjs';
import {shout} from '../lib.mjs';
import {shout} from '/modules/lib.mjs';
import {shout} from 'https://simple.example/modules/lib.mjs';
```

目前，模块说明符必须是完整 URL，或者以`/`,`./`或`../`.

### 默认情况下，模块将延迟 { #defer }

经典`<script>`s 默认阻止 HTML 解析器。您可以通过添加[这`defer`属性](https://html.spec.whatwg.org/multipage/scripting.html#attr-script-defer)，这可确保脚本下载与 HTML 解析并行进行。

![](/\_img/modules/async-defer.svg)

默认情况下，模块脚本处于延迟状态。因此，无需添加`defer`到您的`<script type="module">`标签！不仅主模块的下载与HTML解析并行进行，所有依赖模块也是如此！

## 其他模块功能 { #other功能 }

### 动态`import()`{ #dynamic-import }

到目前为止，我们只使用了静态`import`.带静电`import`，则需要先下载并执行整个模块图，然后才能运行主代码。有时，您不希望预先加载模块，而是按需加载模块，只需在需要时（例如，当用户单击链接或按钮时）。这提高了初始加载时间性能。[动态`import()`](/features/dynamic-import)使这成为可能！

```html
<script type="module">
  (async () => {
    const moduleSpecifier = './lib.mjs';
    const {repeat, shout} = await import(moduleSpecifier);
    repeat('hello');
    // → 'hello hello'
    shout('Dynamic import in action');
    // → 'DYNAMIC IMPORT IN ACTION!'
  })();
</script>
```

与静态不同`import`动态`import()`可以从常规脚本中使用。这是一种在现有代码库中逐步开始使用模块的简单方法。有关更多详细信息，请参阅[我们关于动态的文章`import()`](/features/dynamic-import).

：：：备注
**注意：** [webpack 有自己的版本`import()`](https://developers.google.com/web/fundamentals/performance/webpack/use-long-term-caching)巧妙地将导入的模块拆分为自己的块，与主捆绑包分开。
:::

### `import.meta`{ #import-meta }

另一个与模块相关的新功能是`import.meta`，它为您提供有关当前模块的元数据。您获得的确切元数据未指定为 ECMAScript 的一部分;这取决于主机环境。例如，在浏览器中，您可能会获得与 Node .js 不同的元数据。

下面是一个示例`import.meta`在网络上。默认情况下，图像是相对于 HTML 文档中的当前 URL 加载的。`import.meta.url`使得可以加载相对于当前模块的图像。

```js
function loadThumbnail(relativePath) {
  const url = new URL(relativePath, import.meta.url);
  const image = new Image();
  image.src = url;
  return image;
}

const thumbnail = loadThumbnail('../img/thumbnail.png');
container.append(thumbnail);
```

## 性能建议 { #performance }

### 继续捆绑 { #bundle }

使用模块，可以在不使用捆绑器（如webpack，Rollup或Parcel）的情况下开发网站。在以下场景中直接使用本机 JS 模块是可以的：

*   在本地开发期间
*   对于总共少于 100 个模块且依赖关系树相对较浅（即最大深度小于 5）的小型 Web 应用的生产中

但是，正如我们在[我们在加载由约300个模块组成的模块化库时对Chrome的加载管道的瓶颈分析](https://docs.google.com/document/d/1ovo4PurT\_1K4WFwN2MYmmgbLcr7v6DRQN67ESVA-wq0/pub)，捆绑应用程序的加载性能优于非捆绑应用程序的加载性能。

<figure>
  <a href="https://docs.google.com/document/d/1ovo4PurT_1K4WFwN2MYmmgbLcr7v6DRQN67ESVA-wq0/pub">
    <img src="/_img/modules/renderer-main-thread-time-breakdown.svg" width="830" height="311" alt="" loading="lazy">
  </a>
</figure>

其中一个原因是静态`import`/`export`语法是静态可分析的，因此它可以通过消除未使用的导出来帮助捆绑工具优化代码。静态的`import`和`export`不仅仅是语法;它们是一个关键的工具功能！

*我们的一般建议是在将模块部署到生产环境之前继续使用捆绑程序。*在某种程度上，捆绑是与缩小代码类似的优化：它会带来性能优势，因为您最终会生成更少的代码。捆绑也有同样的效果！继续捆绑。

一如既往，[DevTools Code Coverage 功能](https://developers.google.com/web/updates/2017/04/devtools-release-notes#coverage)可以帮助您识别是否正在向用户推送不必要的代码。我们还建议使用[代码拆分](https://developers.google.com/web/fundamentals/performance/webpack/use-long-term-caching#lazy-loading)拆分捆绑包并推迟加载非“首次有意义”的“绘制”关键脚本。

#### 捆绑与运输非捆绑模块的权衡 { #bundle权衡 }

像往常一样，在Web开发中，一切都是权衡取舍。交付未捆绑模块可能会降低初始加载性能（冷缓存），但与交付单个捆绑包而不进行代码拆分相比，实际上可以提高后续访问（热缓存）的加载性能。对于 200 KB 的代码库，更改单个细粒度模块并将其作为从服务器进行后续访问的唯一提取比必须重新获取整个捆绑包要好得多。

如果您更关心具有热缓存的访问者的体验，而不是首次访问性能，并且站点的细粒度模块少于几百个，则可以尝试使用未捆绑模块，测量冷负载和热负载的性能影响，然后做出数据驱动的决策！

浏览器工程师正在努力提高模块开箱即用的性能。随着时间的推移，我们预计在更多情况下，交付非捆绑模块将变得可行。

### 使用细粒度模块 { #finegrained模块 }

养成使用小型细粒度模块编写代码的习惯。在开发过程中，通常每个模块只有几次导出比手动将大量导出合并到单个文件中更好。

考虑一个名为`./util.mjs`导出三个名为`drop`,`pluck`和`zip`:

```js
export function drop() { /* … */ }
export function pluck() { /* … */ }
export function zip() { /* … */ }
```

如果你的代码库真的只需要`pluck`功能，您可能会按如下方式导入它：

```js
import {pluck} from './util.mjs';
```

在这种情况下，（没有构建时捆绑步骤）浏览器最终仍然必须下载，解析和编译整个`./util.mjs`模块，即使它实际上只需要一个导出。这是浪费！

如果`pluck`不与 共享任何代码`drop`和`zip`，最好将其移动到自己的细粒度模块，例如`./pluck.mjs`.

```js
export function pluck() { /* … */ }
```

然后我们可以导入`pluck`没有处理开销`drop`和`zip`:

```js
import {pluck} from './pluck.mjs';
```

：：：备注
**注意：**您可以使用`default`此处的导出而不是命名导出，具体取决于您的个人喜好。
:::

这不仅可以使您的源代码保持简洁，还可以减少捆绑器执行的死代码消除需求。如果源代码树中的某个模块未使用，则永远不会导入它，因此浏览器永远不会下载它。模块*做*可以单独使用[代码缓存](/blog/code-caching-for-devs)通过浏览器。（实现这一目标的基础设施已经登陆V8，并且[工作正在进行中](https://bugs.chromium.org/p/chromium/issues/detail?id=841466)以在Chrome中启用它。

使用小型、细粒度的模块有助于为将来的代码库做好准备[原生捆绑解决方案](#web-packaging)可能可用。

### 预加载模块 { #preload }

您可以使用以下命令进一步优化模块的交付[`<link rel="modulepreload">`](https://developers.google.com/web/updates/2017/12/modulepreload).这样，浏览器可以预加载甚至准备和预编译模块及其依赖项。

```html
<link rel="modulepreload" href="lib.mjs">
<link rel="modulepreload" href="main.mjs">
<script type="module" src="main.mjs"></script>
<script nomodule src="fallback.js"></script>
```

这对于较大的依赖关系树尤其重要。没有`rel="modulepreload"`，浏览器需要执行多个 HTTP 请求才能找出完整的依赖关系树。但是，如果声明依赖模块脚本的完整列表`rel="modulepreload"`，浏览器不必逐步发现这些依赖项。

### 使用 HTTP/2 { #http2 }

在可能的情况下使用HTTP / 2始终是良好的性能建议，即使仅用于[其多路复用支持](https://developers.google.com/web/fundamentals/performance/http2/#request_and_response_multiplexing).使用 HTTP/2 多路复用，可以同时传输多个请求和响应消息，这有利于加载模块树。

Chrome团队调查了另一个HTTP / 2功能，特别是[HTTP/2 服务器推送](https://developers.google.com/web/fundamentals/performance/http2/#server_push)，可能是部署高度模块化应用程序的实用解决方案。不幸[HTTP/2 服务器推送很难正确](https://jakearchibald.com/2017/h2-push-tougher-than-i-thought/)，并且Web服务器和浏览器的实现目前尚未针对高度模块化的Web应用程序用例进行优化。例如，很难只推送用户尚未缓存的资源，并且通过将源的整个缓存状态传达给服务器来解决隐私风险。

因此，无论如何，请继续使用HTTP / 2！请记住，HTTP / 2服务器推送（不幸的是）不是银弹。

## JS 模块的 Web 采用 { #adoption }

JS模块正在慢慢在网络上得到采用。[我们的使用计数器](https://www.chromestatus.com/metrics/feature/timeline/popularity/2062)显示当前使用的所有页面加载的0.08%`<script type="module">`.请注意，此数字不包括其他入口点，例如动态`import()`或[工件](https://drafts.css-houdini.org/worklets/).

## JS 模块的下一步是什么？{ #next }

Chrome团队正在努力以各种方式改进JS模块的开发时体验。让我们讨论其中的一些。

### 更快、确定性的模块解析算法 { #resolution }

我们建议对模块解析算法进行更改，以解决速度和确定性的缺陷。新算法现已在两者中都可用[HTML 规范](https://github.com/whatwg/html/pull/2991)和[ECMAScript 规范](https://github.com/tc39/ecma262/pull/1006)，并在 中实现[铬 63](http://crbug.com/763597).预计这一改进将很快登陆更多浏览器！

新算法的效率更高，速度更快。旧算法的计算复杂性在依赖关系图的大小上是二次的，即O（n²），Chrome当时的实现也是如此。新算法是线性的，即O（n）。

此外，新算法以确定性方式报告分辨率错误。给定包含多个错误的图形，旧算法的不同运行可能会报告不同的错误，因为它们是导致解决失败的原因。这使得调试变得不必要地困难。新算法保证每次都报告相同的错误。

### Worklets 和 Web workers { #worklets workers }

Chrome 现在实现了[工件](https://drafts.css-houdini.org/worklets/)，它允许Web开发人员在Web浏览器的“低级部分”中自定义硬编码的逻辑。使用worklet，Web开发人员可以将JS模块馈送到渲染管道或音频处理管道中（将来可能会有更多的管道！

Chrome 65 支持[`PaintWorklet`](https://developers.google.com/web/updates/2018/01/paintapi)（又名 CSS Paint API）来控制 DOM 元素的绘制方式。

```js
const result = await css.paintWorklet.addModule('paint-worklet.mjs');
```

Chrome 66 支持[`AudioWorklet`](https://developers.google.com/web/updates/2017/12/audio-worklet)，它允许您使用自己的代码控制音频处理。同一个Chrome版本启动了一个[原产地试用`AnimationWorklet`](https://groups.google.com/a/chromium.org/d/msg/blink-dev/AZ-PYPMS7EA/DEqbe2u5BQAJ)，用于创建滚动链接动画和其他高性能程序动画。

最后[`LayoutWorklet`](https://drafts.css-houdini.org/css-layout-api/)（又名CSS Layout API）现在在Chrome 67中实现。

我们是[加工](https://bugs.chromium.org/p/chromium/issues/detail?id=680046)在 Chrome 中添加对将 JS 模块与专用 Web 工作者配合使用的支持。您已经可以使用以下命令试用此功能`chrome://flags/#enable-experimental-web-platform-features`启用。

```js
const worker = new Worker('worker.mjs', { type: 'module' });
```

对共享工作线程和服务工作线程的 JS 模块支持即将推出：

```js
const worker = new SharedWorker('worker.mjs', { type: 'module' });
const registration = await navigator.serviceWorker.register('worker.mjs', { type: 'module' });
```

### 导入地图 { #package名地图 }

在 Node.js/npm 中，通常使用 JS 模块的“包名称”来导入它们。例如：

```js
import moment from 'moment';
import {pluck} from 'lodash-es';
```

现在[根据 HTML 规范](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)，则此类“裸导入说明符”会引发异常。[我们的进口地图提案](https://github.com/domenic/import-maps)允许此类代码在 Web 上工作，包括在生产应用中。导入映射是一种 JSON 资源，可帮助浏览器将裸导入说明符转换为完整 URL。

导入地图仍处于提案阶段。尽管我们已经对它们如何解决各种用例进行了大量思考，但我们仍在与社区合作，并且尚未编写完整的规范。欢迎反馈！

### Web 打包：本机捆绑包 { #web打包 }

Chrome加载团队目前正在探索[原生网络打包格式](https://github.com/WICG/webpackage)作为分发 Web 应用的新方式。Web打包的核心功能是：

[已签名的 HTTP 交换](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)允许浏览器相信单个HTTP请求/响应对是由它声明的来源生成的;[捆绑的 HTTP 交换](https://tools.ietf.org/html/draft-yasskin-wpack-bundled-exchanges-00)，即交换的集合，每个交换都可以签名或无符号，并具有一些元数据，描述如何将捆绑包作为一个整体进行解释。

综合起来，这样的Web打包格式将实现*多个同源资源*成为*安全嵌入*在*单*断续器`GET`响应。

现有的捆绑工具（如 webpack、Rollup 或 Parcel）当前会发出一个 JavaScript 捆绑包，其中原始独立模块和资源的语义将丢失。使用本机捆绑包，浏览器可以将资源解绑回其原始形式。简单来说，您可以将捆绑的 HTTP Exchange 想象成一个资源捆绑包，可以通过目录（清单）以任何顺序访问这些资源，并且可以根据其相对重要性有效地存储和标记所包含的资源，同时保持单个文件的概念。因此，本机捆绑包可以改善调试体验。在 DevTools 中查看资源时，浏览器可以精确定位原始模块，而无需复杂的源映射。

原生捆绑包格式的透明度开辟了各种优化机会。例如，如果浏览器已经在本地缓存了本机捆绑包的一部分，它可以将其传达给Web服务器，然后仅下载缺少的部分。

Chrome 已经支持该提案的一部分 （[`SignedExchanges`](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)），但捆绑格式本身及其在高度模块化应用程序中的应用仍处于探索阶段。我们非常欢迎您在存储库上或通过电子邮件提供反馈<loading-dev@chromium.org>!

### Layered API { #layered-apis }

发布新功能和 Web API 会产生持续的维护和运行时成本 — 每个新功能都会污染浏览器命名空间，增加启动成本，并代表在整个代码库中引入 bug 的新表面。[分层接口](https://github.com/drufball/layered-apis)是一种努力，以更具可扩展性的方式实现和发布具有Web浏览器的更高级别的API。JS 模块是分层 API 的关键使能技术：

*   由于模块是显式导入的，因此要求通过模块公开分层 API 可确保开发人员只需为他们使用的分层 API 付费。
*   由于模块加载是可配置的，因此分层 API 可以具有内置机制，用于在不支持分层 API 的浏览器中自动加载 polyfill。

模块和分层 API 如何协同工作的详细信息[仍在制定中](https://github.com/drufball/layered-apis/issues)，但当前的提案如下所示：

```html
<script
  type="module"
  src="std:virtual-scroller|https://example.com/virtual-scroller.mjs"
></script>
```

这`<script>`元素加载`virtual-scroller`来自浏览器内置分层 API 集的 API （`std:virtual-scroller`） 或从指向多边形填充的回退 URL。此 API 可以执行 JS 模块在 Web 浏览器中可以执行的任何操作。一个例子是定义[自定义`<virtual-scroller>`元素](https://www.chromestatus.com/feature/5673195159945216)，以便根据需要逐步增强以下 HTML：

```html
<virtual-scroller>
  <!-- Content goes here. -->
</virtual-scroller>
```

## 鸣谢 { #credits }

感谢Domenic Denicola，Georg Neis，Hiroki Nakagawa，Hiroshige Hayashizaki，Jakob Gruber，Kouhei Ueno，Kunihiko Sakamoto和Yang Guo快速制作JavaScript模块！

此外，还要感谢Eric Bidelman，Jake Archibald，Jason Miller，Jeffrey Posnick，Philip Walton，Rob Dodson，Sam Dutton，Sam Thorogood和Thomas Steiner阅读本指南的草稿并给出反馈。
