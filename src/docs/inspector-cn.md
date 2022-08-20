***

## 标题：“在 V8 检查器协议上调试”&#xA;描述：“此页面旨在为嵌入者提供在 V8 中实现调试支持所需的基本工具。

V8 为用户和嵌入器提供了广泛的调试功能。用户通常会通过[Chrome DevTools](https://developer.chrome.com/devtools)接口。嵌入器（包括 DevTools）需要直接依赖于[检查器协议](https://chromedevtools.github.io/debugger-protocol-viewer/tot/).

本页旨在为嵌入者提供在 V8 中实现调试支持所需的基本工具。

## 连接到检查器

V8的[命令行调试外壳程序`d8`](/docs/d8)包括一个简单的检查器集成[`InspectorFrontend`](https://cs.chromium.org/chromium/src/v8/src/d8/d8.cc?l=2286\&rcl=608c4a9c391f3b7cac68068d61f2a8996f216973)和[`InspectorClient`](https://cs.chromium.org/chromium/src/v8/src/d8/d8.cc?l=2355\&rcl=608c4a9c391f3b7cac68068d61f2a8996f216973).客户端为从嵌入器发送到 V8 的消息设置一个通信通道：

```cpp
static void SendInspectorMessage(
    const v8::FunctionCallbackInfo<v8::Value>& args) {
  // [...] Create a StringView that Inspector can understand.
  session->dispatchProtocolMessage(message_view);
}
```

同时，前端通过实现从V8发送到嵌入器的消息建立通道。`sendResponse`和`sendNotification`，然后将其转发到：

```cpp
void Send(const v8_inspector::StringView& string) {
  // [...] String transformations.
  // Grab the global property called 'receive' from the current context.
  Local<String> callback_name =
      v8::String::NewFromUtf8(isolate_, "receive", v8::NewStringType::kNormal)
          .ToLocalChecked();
  Local<Context> context = context_.Get(isolate_);
  Local<Value> callback =
      context->Global()->Get(context, callback_name).ToLocalChecked();
  // And call it to pass the message on to JS.
  if (callback->IsFunction()) {
    // [...]
    MaybeLocal<Value> result = Local<Function>::Cast(callback)->Call(
        context, Undefined(isolate_), 1, args);
  }
}
```

## 使用检查器协议

继续我们的例子，`d8`将检查器消息转发到 JavaScript。以下代码通过以下方式实现与检查器的基本但功能齐全的交互：`d8`:

```js
// inspector-demo.js
// Receiver function called by d8.
function receive(message) {
  print(message)
}

const msg = JSON.stringify({
  id: 0,
  method: 'Debugger.enable',
});

// Call the function provided by d8.
send(msg);

// Run this file by executing 'd8 --enable-inspector inspector-demo.js'.
```

## 更多文档

检查器 API 使用的更充实的示例位于[`test-api.js`](https://cs.chromium.org/chromium/src/v8/test/debugger/test-api.js?type=cs\&q=test-api\&l=1)，它实现了一个简单的调试 API，供 V8 的测试套件使用。

V8 还包含一个替代的检查器集成，位于[`inspector-test.cc`](https://cs.chromium.org/chromium/src/v8/test/inspector/inspector-test.cc?q=inspector-te+package:%5Echromium$\&l=1).

Chrome DevTools wiki 提供了[完整文档](https://chromedevtools.github.io/debugger-protocol-viewer/tot/)所有可用功能。
