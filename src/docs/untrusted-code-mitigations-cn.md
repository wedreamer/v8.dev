***

## 标题： “不受信任的代码缓解措施”&#xA;描述： “如果您嵌入 V8 并运行不受信任的 JavaScript 代码，请启用 V8 的缓解措施以帮助抵御投机性侧信道攻击。

2018年初，谷歌Project Zero的研究人员披露了这一消息。[一类新的攻击](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html)哪[利用](https://security.googleblog.com/2018/01/more-details-about-mitigations-for-cpu\_4.html)许多 CPU 使用的推理执行优化。因为 V8 使用优化的 JIT 编译器 TurboFan 来使 JavaScript 快速运行，所以在某些情况下，它容易受到披露中描述的侧信道攻击。

## 如果仅执行可信代码，则不会有任何变化

如果您的产品仅使用 V8 的嵌入式实例来执行完全受您控制的 JavaScript 或 WebAssembly 代码，那么您对 V8 的使用可能不受推理侧信道攻击 （SSCA） 漏洞的影响。一个只运行您信任的代码的 Node.js实例就是这样一个不受影响的示例。

为了利用此漏洞，攻击者必须在嵌入式环境中执行精心设计的 JavaScript 或 WebAssembly 代码。如果作为开发人员，您可以完全控制嵌入式 V8 实例中执行的代码，那么这不太可能实现。但是，如果您的嵌入式 V8 实例允许下载和执行任意或其他不可信的 JavaScript 或 WebAssembly 代码，甚至生成并随后执行不完全受您控制的 JavaScript 或 WebAssembly 代码（例如，如果它使用其中任何一个作为编译目标），则可能需要考虑缓解措施。

## 如果您确实执行了不受信任的代码...

### 更新到最新的 V8，以便从缓解措施中受益并启用缓解措施

此类攻击的缓解措施在 V8 本身中可用，从[V8 v6.4.388.18](https://chromium.googlesource.com/v8/v8/+/e6eddfe4d1ed9d96b453d14b84ac19769388d8b1)，因此请将 V8 的嵌入式副本更新为[版本6.4.388.18](https://chromium.googlesource.com/v8/v8/+/e6eddfe4d1ed9d96b453d14b84ac19769388d8b1)或稍后建议。旧版本的 V8，包括仍然使用 FullCodeGen 和/或 CrankShaft 的 V8 版本，没有针对 SSCA 的缓解措施。

起价[V8 v6.4.388.18](https://chromium.googlesource.com/v8/v8/+/e6eddfe4d1ed9d96b453d14b84ac19769388d8b1)，V8 中引入了一个新的标志来帮助提供针对 SSCA 漏洞的保护。此标志，称为`--untrusted-code-mitigations`默认情况下，在运行时通过名为`v8_untrusted_code_mitigations`.

这些缓解措施由`--untrusted-code-mitigations`运行时标志：

*   在 WebAssembly 和 asm 中对内存访问之前屏蔽地址.js以确保推理执行的内存加载无法访问 WebAssembly 和 asm.js堆之外的内存。
*   屏蔽 JIT 代码中的索引，用于访问 JavaScript 数组和推理执行路径中的字符串，以确保无法使用数组和字符串到 JavaScript 代码无法访问的内存地址进行推理加载。

嵌入者应注意，缓解措施可能会带来性能权衡。实际影响很大程度上取决于您的工作负载。对于速度计等工作负载，其影响可以忽略不计，但对于更极端的计算工作负载，其影响可能高达15%。如果您完全信任嵌入式 V8 实例执行的 JavaScript 和 WebAssembly 代码，则可以选择通过指定标志来禁用这些 JIT 缓解措施`--no-untrusted-code-mitigations`在运行时。这`v8_untrusted_code_mitigations`GN 标志可用于在构建时启用或禁用缓解措施。

请注意，V8 默认在假定嵌入器将使用进程隔离的平台上禁用这些缓解措施，例如 Chromium 使用站点隔离的平台。

### 在单独的进程中执行沙盒不受信任

如果您在与任何敏感数据分开的进程中执行不受信任的 JavaScript 和 WebAssembly 代码，则 SSCA 的潜在影响将大大降低。通过进程隔离，SSCA 攻击只能观察与执行代码一起在同一进程内沙盒化的数据，而不能观察来自其他进程的数据。

### 考虑调整您提供的高精度计时器

高精度计时器可以更轻松地观察 SSCA 漏洞中的侧信道。如果您的产品提供可由不受信任的 JavaScript 或 WebAssembly 代码访问的高精度计时器，请考虑使这些计时器更加粗糙或增加抖动。
