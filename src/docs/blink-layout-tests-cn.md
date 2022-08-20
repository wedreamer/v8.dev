***

## 标题： “眨眼 Web 测试（又名布局测试）”&#xA;描述： “V8”的基础设施持续运行Blink的Web测试，以防止与Chromium的集成问题。本文档描述了如果此类测试失败该怎么办。

我们不断奔跑[Blink的Web测试（以前称为“布局测试”）](https://chromium.googlesource.com/chromium/src/+/master/docs/testing/web_tests.md)在我们的[集成控制台](https://ci.chromium.org/p/v8/g/integration/console)以防止与铬的集成问题。

在测试失败时，机器人将 V8 树尖的结果与 Chromium 固定的 V8 版本进行比较，仅标记新引入的 V8 问题（误报率< 5%）。责备分配是微不足道的，因为[Linux 版本](https://ci.chromium.org/p/v8/builders/luci.v8.ci/V8%20Blink%20Linux)机器人测试所有修订。

具有新引入的失败的提交通常会还原为取消阻止自动滚动到 Chromium。如果您发现您中断了布局测试，或者您的提交由于此类中断而被还原，并且如果预期更改是预期的，请按照以下步骤在（重新）登陆CL之前将更新的基线添加到Chromium：

1.  登陆铬更改设置`[ Failure Pass ]`对于更改的测试 （[更多](https://chromium.googlesource.com/chromium/src/+/master/docs/testing/web_test_expectations.md#updating-the-expectations-files)).
2.  降落您的V8 CL并等待1-2天，直到它循环进入铬。
3.  跟随[这些说明](https://chromium.googlesource.com/chromium/src/+/master/docs/testing/web_tests.md#Rebaselining-Web-Tests)以手动生成新基线。请注意，如果您仅对铬进行更改，[此首选自动过程](https://chromium.googlesource.com/chromium/src/+/master/docs/testing/web_test_expectations.md#how-to-rebaseline)应该为你工作。
4.  删除`[ Failure Pass ]`从测试期望文件中输入，并将其与Chromium中的新基线一起提交。

请将所有CL与`Bug: …`页脚。
