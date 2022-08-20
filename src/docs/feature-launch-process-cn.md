***

## title： '实现和发布 JavaScript/WebAssembly 语言功能”&#xA;描述： “本文档解释了在 V8 中实现和发布 JavaScript 或 WebAssembly 语言特性的过程。

通常，V8 使用[眨眼意图过程](https://www.chromium.org/blink/launching-features)用于 JavaScript 和 WebAssembly 语言功能。这些差异在下面的勘误表中列出。请遵循眨眼意图过程，除非勘误表另有说明。

如果您对此主题有任何疑问，请发送电子邮件 hablich@chromium.org 并 v8-dev@googlegroups.com。

## 勘误表

### TAG评论 { #tag }

对于较小的JavaScript或WebAssembly功能，不需要TAG审查，因为TC39和Wasm CG已经提供了重要的技术监督。如果功能很大或横切（例如，需要更改其他Web平台API或修改Chromium），建议进行TAG审查。

### 代替WPT，Test262和WebAssembly规范测试就足够了{ #tests }

添加Web平台测试（WPT）不是必需的，因为JavaScript和WebAssembly语言功能有自己的测试存储库。不过，如果您认为它是有益的，请随时添加一些。

对于 JavaScript 功能，在[测试262](https://github.com/tc39/test262)是首选和必需的。

对于 WebAssembly 功能，在[WebAssembly Spec Test repo](https://github.com/WebAssembly/spec/tree/master/test)是必需的。

### 抄送谁 { #cc }

**每**“意向`$something`“电子邮件（例如”实施意图“）应抄送<v8-users@googlegroups.com>除了<blink-dev@chromium.org>.这样，V8的其他嵌入器也保持在循环中。

### 链接到规范存储库 { #spec }

眨眼意图过程需要一个解释器。无需编写新文档，而是随意链接到相应的规范存储库（例如[`import.meta`](https://github.com/tc39/proposal-import-meta)).
