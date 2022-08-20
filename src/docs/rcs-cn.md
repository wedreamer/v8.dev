***

## 标题： '运行时调用统计'&#xA;描述：“本文档介绍如何使用运行时调用统计信息来获取详细的 V8 内部指标。

[“开发工具性能”面板](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/)通过可视化各种 Chrome 内部指标，深入了解网络应用的运行时性能。但是，某些低级 V8 指标当前未在 DevTools 中公开。本文将指导您完成收集详细的 V8 内部指标（称为运行时调用统计信息或 RCS）的最可靠方法，通过`chrome://tracing`.

跟踪会记录整个浏览器（包括其他选项卡、窗口和扩展程序）的行为，因此，在干净的用户配置文件中完成此操作、禁用扩展程序且未打开其他浏览器选项卡时，跟踪效果最佳：

```bash
# Start a new Chrome browser session with a clean user profile and extensions disabled
google-chrome --user-data-dir="$(mktemp -d)" --disable-extensions
```

在第一个选项卡中键入要测量的页面的 URL，但尚未加载该页面。

![](/\_img/rcs/01.png)

添加第二个标签页并打开`chrome://tracing`.提示：您可以直接输入`chrome:tracing`，不带斜杠。

![](/\_img/rcs/02.png)

单击“记录”按钮以准备记录跟踪。首先选择“Web开发人员”，然后选择“编辑类别”。

![](/\_img/rcs/03.png)

选择`v8.runtime_stats`从列表中。根据调查的详细程度，您也可以选择其他类别。

![](/\_img/rcs/04.png)

按“录制”并切换回第一个选项卡并加载页面。最快的方法是使用<kbd>Ctrl</kbd>/<kbd>⌘</kbd>+<kbd>1</kbd>直接跳转到第一个选项卡，然后按<kbd>进入</kbd>以接受输入的 URL。

![](/\_img/rcs/05.png)

等到页面加载完成或缓冲区已满，然后“停止”录制。

![](/\_img/rcs/06.png)

从录制的选项卡中查找包含网页标题的“呈现器”部分。最简单的方法是单击“进程”，然后单击“无”以取消选中所有条目，然后仅选择您感兴趣的渲染器。

![](/\_img/rcs/07.png)

选择跟踪事件/切片，方法是按<kbd>转变</kbd>和拖动。确保覆盖*都*各节，包括`CrRendererMain`和任何`ThreadPoolForegroundWorker`s.底部将显示一个包含所有选定切片的表格。

![](/\_img/rcs/08.png)

滚动到表格的右上角，然后单击“运行时调用统计信息表格”旁边的链接。

![](/\_img/rcs/09.png)

在显示的视图中，滚动到底部以查看 V8 花费时间的详细表格。

![](/\_img/rcs/10.png)

通过翻转打开一个类别，您可以进一步向下钻取到数据。

![](/\_img/rcs/11.png)

## 命令行界面 { #cli }

跑[`d8`](/docs/d8)跟`--runtime-call-stats`从命令行获取 RCS 指标：

```bash
d8 --runtime-call-stats foo.js
```
