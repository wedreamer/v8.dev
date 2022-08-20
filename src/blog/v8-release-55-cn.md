***

标题： 'V8 版本 v5.5'
作者： 'V8团队'
日期： 2016-10-24 13：33：37
标签：

*   释放
    描述： 'V8 v5.5 减少了内存消耗，并增加了对 ECMAScript 语言功能的支持。

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 5.5](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/5.5)，它将处于测试阶段，直到几周后与Chrome 55 Stable协同发布。V8 v5.5 充满了各种面向开发人员的好东西，所以我们想在发布之前预览一些亮点。

## 语言功能

### 异步函数

在 v5.5 中，V8 发布了 JavaScript ES2017[异步函数](https://developers.google.com/web/fundamentals/getting-started/primers/async-functions)，这使得编写使用和创建 Promise 的代码变得更加容易。使用异步函数，等待 Promise 解析就像在它之前键入 await 一样简单，然后像值同步可用一样继续 - 不需要回调。看[本文](https://developers.google.com/web/fundamentals/getting-started/primers/async-functions)介绍。

下面是一个示例函数，它提取 URL 并返回响应的文本，该文本以典型的异步、基于 Promise 的样式编写。

```js
function logFetch(url) {
  return fetch(url)
    .then(response => response.text())
    .then(text => {
      console.log(text);
    }).catch(err => {
      console.error('fetch failed', err);
    });
}
```

下面是使用异步函数重写以删除回调的相同代码。

```js
async function logFetch(url) {
  try {
    const response = await fetch(url);
    console.log(await response.text());
  } catch (err) {
    console.log('fetch failed', err);
  }
}
```

## 性能改进

V8 v5.5 在内存占用方面提供了许多关键改进。

### 记忆

内存消耗是 JavaScript 虚拟机性能权衡空间中的一个重要维度。在过去的几个版本中，V8 团队分析并显著减少了几个网站的内存占用，这些网站被确定为现代 Web 开发模式的代表。V8 5.5 将 Chrome 的整体内存消耗降低了 35%**低内存设备**（与 Chrome 53 中的 V8 5.3 相比），由于 V8 堆大小和区域内存使用量的减少。其他设备段也受益于区域内存减少。请看一下[专门的博客文章](/blog/optimizing-v8-memory)以获取详细视图。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

### V8 检查器已迁移

V8检查器从Chromium迁移到V8。检查器代码现在完全驻留在[V8 存储库](https://chromium.googlesource.com/v8/v8/+/master/src/inspector/).

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 5.5 -t branch-heads/5.5`以试验 V8 5.5 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
