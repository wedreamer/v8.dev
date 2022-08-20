***

标题： '动态`import()`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2017-11-21
    标签：
*   ECMAScript
*   ES2020
    描述：与静态导入相比，“动态导入（）”解锁了新功能。本文比较了两者，并概述了新功能。
    推文：“932914724060254208”

***

[动态`import()`](https://github.com/tc39/proposal-dynamic-import)引入了一种新的类似函数的形式`import`与静态相比，可解锁新功能`import`.本文比较了两者，并概述了新增功能。

## 静态的`import`（回顾）{ #static }

Chrome 61 附带了对 ES2015 的支持`import`语句内[模块](/features/modules).

请考虑以下模块，位于`./utils.mjs`:

```js
// Default export
export default () => {
  console.log('Hi from the default export!');
};

// Named export `doStuff`
export const doStuff = () => {
  console.log('Doing stuff…');
};
```

下面介绍了如何静态导入和使用`./utils.mjs`模块：

```html
<script type="module">
  import * as module from './utils.mjs';
  module.default();
  // → logs 'Hi from the default export!'
  module.doStuff();
  // → logs 'Doing stuff…'
</script>
```

：：：备注
**注意：**前面的示例使用`.mjs`扩展以表明它是一个模块而不是常规脚本。在网络上，文件扩展名并不重要，只要文件以正确的MIME类型提供（例如`text/javascript`对于 JavaScript 文件），在`Content-Type`HTTP 标头。

这`.mjs`扩展在其他平台上特别有用，例如[节点.js](https://nodejs.org/api/esm.html#esm_enabling)和[`d8`](/docs/d8)，其中没有 MIME 类型或其他强制挂钩的概念，例如`type="module"`以确定某些内容是模块还是常规脚本。我们在这里使用相同的扩展，以实现跨平台的一致性，并清楚地区分模块和常规脚本。
:::

这种用于导入模块的语法形式是*静态的*声明：它只接受一个字符串文本作为模块说明符，并通过运行时前的“链接”过程将绑定引入本地范围。静态`import`语法只能在文件的顶层使用。

静态的`import`支持重要的用例，如静态分析、捆绑工具和树摇动。

在某些情况下，执行以下操作很有用：

*   按需（或有条件）导入模块
*   在运行时计算模块说明符
*   从常规脚本（而不是模块）中导入模块

这些都不是静态的`import`.

## 动态`import()`🔥 { #dynamic }

[动态`import()`](https://github.com/tc39/proposal-dynamic-import)引入了一种新的类似函数的形式`import`迎合这些用例。`import(moduleSpecifier)`返回所请求模块的模块命名空间对象的承诺，该承诺是在获取、实例化和评估模块的所有依赖项以及模块本身之后创建的。

下面介绍如何动态导入和使用`./utils.mjs`模块：

```html
<script type="module">
  const moduleSpecifier = './utils.mjs';
  import(moduleSpecifier)
    .then((module) => {
      module.default();
      // → logs 'Hi from the default export!'
      module.doStuff();
      // → logs 'Doing stuff…'
    });
</script>
```

因为`import()`返回承诺，可以使用`async`/`await`而不是`then`基于回调样式：

```html
<script type="module">
  (async () => {
    const moduleSpecifier = './utils.mjs';
    const module = await import(moduleSpecifier)
    module.default();
    // → logs 'Hi from the default export!'
    module.doStuff();
    // → logs 'Doing stuff…'
  })();
</script>
```

：：：备注
**注意：**虽然`import()` *看起来*就像函数调用一样，它被指定为*语法*只是碰巧使用括号（类似于[`super()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)).这意味着`import`不继承自`Function.prototype`所以你不能`call`或`apply`它，以及诸如此类的东西`const importAlias = import`不要工作 - 哎呀，`import`甚至不是一个物体！不过，这在实践中并不重要。
:::

下面是动态性的示例`import()`在小型单页应用程序中导航时启用延迟加载模块：

```html
<!DOCTYPE html>
<meta charset="utf-8">
<title>My library</title>
<nav>
  <a href="books.html" data-entry-module="books">Books</a>
  <a href="movies.html" data-entry-module="movies">Movies</a>
  <a href="video-games.html" data-entry-module="video-games">Video Games</a>
</nav>
<main>This is a placeholder for the content that will be loaded on-demand.</main>
<script>
  const main = document.querySelector('main');
  const links = document.querySelectorAll('nav > a');
  for (const link of links) {
    link.addEventListener('click', async (event) => {
      event.preventDefault();
      try {
        const module = await import(`/${link.dataset.entryModule}.mjs`);
        // The module exports a function named `loadPageInto`.
        module.loadPageInto(main);
      } catch (error) {
        main.textContent = error.message;
      }
    });
  }
</script>
```

动态启用的延迟加载功能`import()`如果应用得当，可以非常强大。出于演示目的，[艾迪](https://twitter.com/addyosmani)改 性[一个例子 黑客新闻 PWA](https://hnpwa-vanilla.firebaseapp.com/)在首次加载时静态导入其所有依赖项，包括注释。[更新的版本](https://dynamic-import.firebaseapp.com/)使用动态`import()`懒洋洋地加载注释，避免加载，解析和编译成本，直到用户真正需要它们。

：：：备注
**注意：**如果你的应用从其他域（静态或动态）导入脚本，则需要使用有效的 CORS 标头返回脚本（例如`Access-Control-Allow-Origin: *`).这是因为与常规脚本不同，模块脚本（及其导入）是使用 CORS 获取的。
:::

## 建议

静态的`import`和动态`import()`都很有用。每个都有自己非常独特的用例。使用静态`import`s 表示初始绘制依赖关系，尤其是对于首屏内容。在其他情况下，请考虑使用动态按需加载依赖项`import()`.

## 动态`import()`支持 { #support }

<feature-support chrome="63"
              firefox="67"
              safari="11.1"
              nodejs="13.2 https://nodejs.medium.com/announcing-core-node-js-support-for-ecmascript-modules-c5d6dc29b663"
              babel="yes https://babeljs.io/docs/en/babel-plugin-syntax-dynamic-import"></feature-support>
