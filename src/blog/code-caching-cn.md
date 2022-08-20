***

标题： “代码缓存”
作者： '杨果 （[@hashseed](https://twitter.com/hashseed)）， 软件工程师
化身：

*   “阳国”
    日期： 2015-07-27 13：33：37
    标签：
*   内部
    描述： 'V8 现在支持（字节）代码缓存，即缓存 JavaScript 解析 + 编译的结果。

***

V8 用途[即时编译](https://en.wikipedia.org/wiki/Just-in-time_compilation)（JIT） 来执行 JavaScript 代码。这意味着在运行脚本之前，必须立即对其进行解析和编译 - 这可能会导致相当大的开销。正如我们[最近宣布](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html)，代码缓存是一种减少此开销的技术。首次编译脚本时，将生成并存储缓存数据。下次 V8 需要编译相同的脚本时，即使在不同的 V8 实例中，它也可以使用缓存数据来重新创建编译结果，而不是从头开始编译。因此，脚本的执行速度要快得多。

代码缓存自 V8 版本 4.2 起可用，并且不仅限于 Chrome。它通过 V8 的 API 公开，因此每个 V8 嵌入器都可以利用它。这[测试用例](https://chromium.googlesource.com/v8/v8.git/+/4.5.56/test/cctest/test-api.cc#21090)用于执行此功能可作为如何使用此 API 的示例。

当脚本由 V8 编译时，可以生成缓存数据，通过传递来加快以后的编译速度`v8::ScriptCompiler::kProduceCodeCache`作为选项。如果编译成功，缓存数据将附加到源对象，并可通过`v8::ScriptCompiler::Source::GetCachedData`.然后可以将其持久保存以供以后使用，例如通过将其写入磁盘。

在以后的编译过程中，可以将以前生成的缓存数据附加到源对象并传递`v8::ScriptCompiler::kConsumeCodeCache`作为选项。这一次，代码的生成速度将快得多，因为 V8 绕过了编译代码，并从提供的缓存数据中反序列化了代码。

生成缓存数据需要一定的计算和内存成本。因此，Chrome 只有在几天内至少看到同一脚本两次时，才会生成缓存数据。通过这种方式，Chrome能够以平均两倍的速度将脚本文件转换为可执行代码，从而为用户在每次后续页面加载时节省宝贵的时间。
