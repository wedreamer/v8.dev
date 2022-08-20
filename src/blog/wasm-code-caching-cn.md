***

标题： 'WebAssembly 开发人员的代码缓存'
作者： '[比尔·巴奇](https://twitter.com/billb)，把卡青！在缓存中'
化身：

*   账单让步
    日期： 2019-06-17
    标签：
*   WebAssembly
*   内部
    描述： “本文解释了Chrome的WebAssembly代码缓存，以及开发人员如何利用它来加速加载具有大型WebAssembly模块的应用程序。
    推文：“1140631433532334081”

***

开发人员中有一种说法，最快的代码是不运行的代码。同样，编译代码最快的是不必编译的代码。WebAssembly代码缓存是Chrome和V8中的一项新优化，它试图通过缓存编译器生成的本机代码来避免代码编译。我们已经[写](/blog/code-caching) [大约](/blog/improved-code-caching) [如何](/blog/code-caching-for-devs)Chrome 和 V8 过去缓存 JavaScript 代码，以及利用此优化的最佳实践。在这篇博客文章中，我们描述了Chrome的WebAssembly代码缓存的操作，以及开发人员如何利用它来加速具有大型WebAssembly模块的应用程序的加载速度。

## WebAssembly 编译回顾 { #recap }

WebAssembly是一种在Web上运行非JavaScript代码的方法。Web 应用程序可以通过加载`.wasm`resource，其中包含来自其他语言的部分编译代码，例如 C、C++ 或 Rust（以及更多即将推出的代码）。WebAssembly编译器的工作是解码`.wasm`资源，验证它是否格式正确，然后将其编译为可在用户计算机上执行的本机代码。

V8有两个WebAssembly编译器：Liftoff和TurboFan。[起飞](/blog/liftoff)是基线编译器，它尽可能快地编译模块，以便可以尽快开始执行。TurboFan是V8针对JavaScript和WebAssembly的优化编译器。它在后台运行以生成高质量的本机代码，从而为 Web 应用提供长期最佳性能。对于大型WebAssembly模块，TurboFan可能需要大量时间（30秒到一分钟或更长时间）才能完全完成将WebAssembly模块编译为本机代码。

这就是代码缓存的用武之地。一旦TurboFan完成了对大型WebAssembly模块的编译，Chrome就可以将代码保存在其缓存中，以便下次加载模块时，我们可以跳过Liftoff和TurboFan编译，从而加快启动速度并降低功耗 - 编译代码非常占用CPU。

WebAssembly代码缓存在Chrome中使用与JavaScript代码缓存相同的机制。我们使用相同类型的存储，以及相同的双键缓存技术，该技术将不同来源编译的代码保持独立，符合[站点隔离](https://developers.google.com/web/updates/2018/07/site-isolation)，这是一项重要的 Chrome 安全功能。

## WebAssembly code caching algorithm { #algorithm }

目前，WebAssembly缓存仅用于流式API调用，`compileStreaming`和`instantiateStreaming`.这些操作在 HTTP 获取`.wasm`资源，使使用Chrome的资源获取和缓存机制变得更加容易，并提供一个方便的资源URL，用作识别WebAssembly模块的关键。缓存算法的工作原理如下：

1.  当`.wasm`首先请求资源（即*冷运行*），Chrome 从网络下载它并将其流式传输到 V8 进行编译。Chrome 还存储`.wasm`资源在浏览器的资源缓存中，存储在用户设备的文件系统中。借助此资源缓存，Chrome 可以在下次需要时更快地加载资源。
2.  当 TurboFan 完全完成对模块的编译时，如果`.wasm`资源足够大（目前为128 kB），Chrome将编译后的代码写入WebAssembly代码缓存。此代码缓存在物理上独立于步骤 1 中的资源缓存。
3.  当`.wasm`第二次请求资源（即*热运行*），Chrome 会加载`.wasm`资源，并同时查询代码缓存。如果缓存命中，则编译的模块字节将发送到渲染器进程并传递到 V8，后者将反序列化代码而不是编译模块。反序列化比编译更快，CPU 占用更少。
4.  可能是缓存的代码不再有效。发生这种情况可能是因为`.wasm`资源已经改变，或者因为V8已经改变，由于Chrome的快速发布周期，预计至少每6周就会发生一次。在这种情况下，将从缓存中清除缓存的本机代码，并按照步骤 1 中的步骤进行编译。

根据此描述，我们可以提供一些建议，以改善您的网站对WebAssembly代码缓存的使用。

## 提示 1：使用 WebAssembly 流式处理 API { #stream }

由于代码缓存仅适用于流式 API，因此请使用以下命令编译或实例化 WebAssembly 模块`compileStreaming`或`instantiateStreaming`，如下面的 JavaScript 片段所示：

```js
(async () => {
  const fetchPromise = fetch('fibonacci.wasm');
  const { instance } = await WebAssembly.instantiateStreaming(fetchPromise);
  const result = instance.exports.fibonacci(42);
  console.log(result);
})();
```

这[品](https://developers.google.com/web/updates/2018/04/loading-wasm)详细介绍了使用WebAssembly流媒体API的优势。默认情况下，Emscripten 在为应用生成加载程序代码时会尝试使用此 API。请注意，流式传输要求`.wasm`资源具有正确的 MIME 类型，因此服务器必须发送`Content-Type: application/wasm`标头在其响应中。

## 提示 2：对缓存友好 { #cache友好 }

由于代码缓存取决于资源 URL 以及`.wasm`资源是最新的，开发人员应该尽量保持这两者的稳定。如果`.wasm`resource是从不同的URL获取的，它被认为是不同的，V8必须再次编译模块。同样，如果`.wasm`资源在资源缓存中不再有效，然后Chrome必须丢弃所有缓存的代码。

### 保持代码稳定 { #keep代码稳定 }

每当您发布新的WebAssembly模块时，都必须完全重新编译它。仅在需要交付新功能或修复 Bug 时才发布新版本的代码。如果您的代码未发生更改，请告知 Chrome。当浏览器对资源 URL（如 WebAssembly 模块）发出 HTTP 请求时，它会包含上次获取该 URL 的日期和时间。如果服务器知道文件未更改，它可以发回`304 Not Modified`响应，它告诉 Chrome 和 V8 缓存的资源以及缓存的代码仍然有效。另一方面，返回`200 OK`响应更新缓存的`.wasm`资源，并使代码缓存失效，将 WebAssembly 恢复为冷运行。跟随[网络资源最佳实践](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)通过使用响应通知浏览器是否`.wasm`资源是可缓存的，它预计有效多长时间，或者上次修改的时间。

### 不要更改代码的 URL { #url }

缓存的已编译代码与`.wasm`资源，这使得查找变得容易，而不必扫描实际资源。这意味着更改资源的 URL（包括任何查询参数！）会在资源缓存中创建一个新条目，这还需要完全重新编译并创建新的代码缓存条目。

### 做大（但不要太大！{ #go-大 }

WebAssembly代码缓存的主要启发式方法是`.wasm`资源。如果`.wasm`资源小于某个阈值大小，我们不缓存已编译的模块字节。这里的理由是，V8可以快速编译小模块，可能比从缓存加载编译的代码更快。目前，截止时间是`.wasm`资源为 128 kB 或更多。

但只有在一定程度上，越大越好。由于缓存会占用用户计算机上的空间，因此 Chrome 会小心翼翼地避免占用太多空间。现在，在台式计算机上，代码缓存通常保存几百兆字节的数据。由于Chrome缓存还将缓存中的最大条目限制为总缓存大小的一小部分，因此编译的WebAssembly代码还有大约150 MB的进一步限制（总缓存大小的一半）。重要的是要注意，编译的模块通常比相应的模块大5-7倍`.wasm`典型桌面计算机上的资源。

与缓存行为的其余部分一样，此大小启发式可能会在我们确定最适合用户和开发人员的行为时发生变化。

### 使用服务工作线程 { #service工作线程 }

WebAssembly 代码缓存已为辅助角色和服务工作线程启用，因此可以使用它们来加载、编译和缓存新版本的代码，以便在下次应用启动时可以使用它。每个网站必须至少执行一次WebAssembly模块的完整编译 - 使用worker向用户隐藏它。

## 描图

作为开发者，您可能需要检查 Chrome 是否缓存了已编译的模块。默认情况下，WebAssembly代码缓存事件不会在Chrome的开发人员工具中公开，因此找出您的模块是否正在缓存的最佳方法是使用略低级别`chrome://tracing`特征。

`chrome://tracing`记录在一段时间内检测的 Chrome 痕迹。跟踪会记录整个浏览器（包括其他选项卡、窗口和扩展程序）的行为，因此，在干净的用户配置文件中完成此操作、禁用扩展程序且未打开其他浏览器选项卡时，跟踪效果最佳：

```bash
# Start a new Chrome browser session with a clean user profile and extensions disabled
google-chrome --user-data-dir="$(mktemp -d)" --disable-extensions
```

导航到`chrome://tracing`，然后单击“记录”以开始跟踪会话。在出现的对话框窗口中，单击“编辑类别”，然后选中`devtools.timeline`“默认类别禁用”下右侧的类别（您可以取消选中任何其他预选类别以减少收集的数据量）。然后单击对话框中的“记录”按钮开始跟踪。

在另一个选项卡中加载或重新加载您的应用。让它运行足够长的时间，10秒或更长时间，以确保TurboFan编译完成。完成后，单击“停止”以结束跟踪。此时将显示事件的时间线视图。在跟踪窗口的右上角，有一个文本框，就在“查看选项”的右侧。类型`v8.wasm`以过滤掉非 WebAssembly 事件。您应看到以下一个或多个事件：

*   `v8.wasm.streamFromResponseCallback`— 传递给实例化流的资源获取收到了响应。
*   `v8.wasm.compiledModule`— TurboFan 已完成编译`.wasm`资源。
*   `v8.wasm.cachedModule`- Chrome将编译后的模块写入代码缓存。
*   `v8.wasm.moduleCacheHit`- Chrome在加载时在其缓存中找到了该代码`.wasm`资源。
*   `v8.wasm.moduleCacheInvalid`— V8 无法反序列化缓存的代码，因为它已过期。

在冷跑中，我们希望看到`v8.wasm.streamFromResponseCallback`和`v8.wasm.compiledModule`事件。这表示 WebAssembly 模块已收到，编译成功。如果两个事件都没有观察到，请检查 WebAssembly 流式 API 调用是否正常工作。

冷运行后，如果超过大小阈值，我们还希望看到`v8.wasm.cachedModule`事件，表示已编译的代码已发送到缓存。我们可能收到此事件，但由于某种原因，写入未成功。目前没有办法观察到这一点，但事件的元数据可以显示代码的大小。非常大的模块可能不适合缓存。

当缓存正常工作时，热运行会生成两个事件：`v8.wasm.streamFromResponseCallback`和`v8.wasm.moduleCacheHit`.通过这些事件的元数据，您可以查看已编译代码的大小。

有关使用的更多信息`chrome://tracing`看[我们关于开发人员的JavaScript（字节）代码缓存的文章](/blog/code-caching-for-devs).

## 结论

对于大多数开发人员来说，代码缓存应该“正常工作”。当事情稳定时，它像任何缓存一样效果最好。Chrome的缓存启发式方法可能会在版本之间发生变化，但代码缓存确实具有可以使用的行为以及可以避免的限制。仔细分析使用`chrome://tracing`可以帮助您调整和优化Web应用程序对WebAssembly代码缓存的使用。
