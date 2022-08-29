***

标题： 'Indicium： V8 运行时跟踪工具'
作者： 'Zeynep Cankara （[@ZeynepCankara](https://twitter.com/ZeynepCankara))'
化身：

*   'zeynep-cankara'
    日期： 2020-10-01 11：56：00
    标签：
*   工具
*   系统分析仪
    描述： 'Indicium：用于分析地图/IC 事件的 V8 系统分析器工具。
    推文：“1311689392608731140”

***

# 标志：V8 系统分析仪

过去的三个月对我来说是一次很棒的学习经历，因为我作为实习生加入了V8团队（谷歌伦敦），并一直在研究一种名为[*印地铟*](https://v8.dev/tools/head/system-analyzer).

该系统分析器是一个统一的Web界面，用于跟踪，调试和分析如何在实际应用程序中创建和修改内联缓存（IC）和地图的模式。

V8 已经有一个跟踪基础结构，用于[集成电路](https://mathiasbynens.be/notes/shapes-ics)和[地图](https://v8.dev/blog/fast-properties)它可以使用[集成电路浏览器](https://v8.dev/tools/v8.7/ic-explorer.html)和映射事件使用[地图处理器](https://v8.dev/tools/v8.7/map-processor.html).然而，以前的工具不允许我们全面分析地图和IC，现在系统分析仪可以做到这一点。

![Indicium](../_img/system-analyzer/indicium-logo.png)

## 个案研究

让我们通过一个示例来演示如何使用 Indicium 来分析 V8 中的 Map 和 IC 日志事件。

```javascript
class Point {
  constructor(x, y) {
    if (x < 0 || y < 0) {
      this.isNegative = true;
    }
    this.x = x;
    this.y = y;
  }

  dotProduct(other) {
    return this.x * other.x + this.y * other.y;
  }
}

let a = new Point(1, 1);
let b = new Point(2, 2);
let dotProduct;

// warmup
for (let i = 0; i < 10e5; i++) {
  dotProduct = a.dotProduct(b);
}

console.time('snippet1');
for (let i = 0; i < 10e6; i++) {
  dotProduct = a.dotProduct(b);
}
console.timeEnd('snippet1');

a = new Point(-1, -1);
b = new Point(-2, -2);
console.time('snippet2');
for (let i = 0; i < 10e6; i++) {
  dotProduct = a.dotProduct(b);
}
console.timeEnd('snippet2');
```

在这里，我们有一个`Point`类，它存储两个坐标和一个基于坐标值的附加布尔值。这`Point`类有一个`dotProduct`返回传递的对象和接收方之间的点积的方法。

为了更容易地解释程序，让我们将程序分成两个片段（忽略预热阶段）：

### *片段 1*

```javascript
let a = new Point(1, 1);
let b = new Point(2, 2);
let dotProduct;

console.time('snippet1');
for (let i = 0; i < 10e6; i++) {
  dotProduct = a.dotProduct(b);
}
console.timeEnd('snippet1');
```

### *片段 2*

```javascript
a = new Point(-1, -1);
b = new Point(-2, -2);
console.time('snippet2');
for (let i = 0; i < 10e6; i++) {
  dotProduct = a.dotProduct(b);
}
console.timeEnd('snippet2');
```

运行程序后，我们会注意到性能下降。即使我们正在测量两个类似片段的性能;访问属性`x`和`y`之`Point`通过调用`dotProduct`函数在 for 循环中。

代码段 1 的运行速度比代码段 2 快约 3 倍。唯一的区别是我们使用负值`x`和`y`属性`Point`代码段 2 中的对象。

![Performance analysis of snippets.](../_img/system-analyzer/initial-program-performance.png)

为了分析这种性能差异，我们可以使用 V8 附带的各种日志记录选项。这就是系统分析仪的亮点。它可以显示日志事件，并将它们与地图事件链接在一起，让我们探索隐藏在V8中的魔力。

在深入研究案例研究之前，让我们先熟悉一下系统分析器工具的面板。该工具有四个主要面板：

*   一个时间轴面板，用于分析跨时间的Map / IC事件，
*   一个地图面板，用于可视化地图的过渡树，
*   一个IC面板，用于获取有关IC事件的统计信息，
*   一个“源”面板，用于在脚本上显示地图/IC 文件位置。

![System Analyzer Overview](../_img/system-analyzer/system-analyzer-overview.png)

![Group IC events by function name to get in depth information about the IC events associated with the dotProduct.](../_img/system-analyzer/case1\_1.png)

我们正在分析函数`dotProduct`可能会导致此性能差异。因此，我们按函数名称对 IC 事件进行分组，以获取有关与`dotProduct`功能。

我们注意到的第一件事是，此函数中的 IC 事件记录了两个不同的 IC 状态转换。一个从未初始化到单态，另一个从单态到多态。多态IC状态表示现在我们正在跟踪多个与`Point`对象和这种多态状态更糟，因为我们必须执行额外的检查。

我们想知道为什么要为同一类型的对象创建多个 Map 形状。为此，我们切换有关 IC 状态的信息按钮，以获取有关 Map 地址从未初始化到单态的更多信息。

![The map transition tree associated with the monomorphic IC state.](../_img/system-analyzer/case1\_2.png)

![The map transition tree associated with the polymorphic IC state.](../_img/system-analyzer/case1\_3.png)

对于单态 IC 状态，我们可以可视化过渡树，并看到我们只是动态添加两个属性`x`和`y`但是当涉及到多态IC状态时，我们有一个包含三个属性的新Map。`isNegative`,`x`和`y`.

![The Map panel communicates the file position information to highlight file positions on the Source panel.](../_img/system-analyzer/case1\_4.png)

我们单击“地图”面板的文件位置部分以查看此位置`isNegative`属性添加到源代码中，可以使用此见解来解决性能回归问题。

所以现在的问题是*我们如何通过使用我们从工具中生成的见解来解决性能倒退问题*?

最小的解决方案是始终初始化`isNegative`财产。通常，建议所有实例属性都应在构造函数中初始化。

现在，更新了`Point`类看起来像这样：

```javascript
class Point {
  constructor(x, y) {
    this.isNegative = x < 0 || y < 0;
    this.x = x;
    this.y = y;
  }

  dotProduct(other) {
    return this.x * other.x + this.y * other.y;
  }
}
```

如果我们再次执行脚本并修改`Point`类，我们看到在案例研究开始时定义的两个片段的执行执行非常相似。

在更新的迹线中，我们看到多态IC状态被避免了，因为我们没有为同一类型的对象创建多个映射。

![The map transition tree of the modified Point object.](../_img/system-analyzer/case2\_1.png)

## 系统分析器

现在，让我们深入了解一下系统分析器中存在的不同面板。

### 时间轴面板

“时间轴”面板允许按时间进行选择，从而可以跨离散时间点或选定时间范围的 IC/映射状态进行可视化。它支持过滤功能，例如放大/缩小所选时间范围的日志事件。

![Timeline panel overview](../_img/system-analyzer/timeline-panel.png)

![Timeline panel overview (Cont.)](../_img/system-analyzer/timeline-panel2.png)

### 地图面板

“地图”面板有两个子面板：

1.  地图详情
2.  地图过渡

“地图”面板可显示所选地图的过渡树。通过地图详细信息子面板显示的所选地图的元数据。可以使用提供的界面搜索与映射地址关联的特定过渡树。从“地图过渡”子面板上方的“统计信息”子面板中，我们可以看到有关导致地图过渡的属性和地图事件类型的统计信息。

![Map panel overview](../_img/system-analyzer/map-panel.png)

![Stats panel overview](../_img/system-analyzer/stats-panel.png)

### 集成电路面板

IC 面板显示有关特定时间范围内的 IC 事件的统计信息，这些统计信息通过“时间轴”面板进行过滤。此外，IC面板允许根据各种选项（类型，类别，地图，文件位置）对IC事件进行分组。在分组选项中，地图和文件位置分组选项分别与地图和源代码面板交互，以显示地图的过渡树并突出显示与 IC 事件关联的文件位置。

![IC panel Overview](../_img/system-analyzer/ic-panel.png)

![IC panel overview (Cont.)](../_img/system-analyzer/ic-panel2.png)

![IC panel Overview (Cont.)](../_img/system-analyzer/ic-panel3.png)

![IC panel overview (Cont.)](../_img/system-analyzer/ic-panel4.png)

### “源”面板

“源”面板显示加载的脚本，并带有可单击的标记，以发出自定义事件，从而在自定义面板中选择映射和 IC 日志事件。可以从向下钻取栏选择加载的脚本。从“地图”面板和 IC 面板中选择文件位置会在源代码面板上突出显示所选文件位置。

![Source panel Overview](../_img/system-analyzer/source-panel.png)

### 确认

我要感谢V8和Web中Android团队的每个人，特别是我的主持人Sathya和共同主持人Camillo，感谢他们在实习期间为我提供支持，并让我有机会参与这样一个很酷的项目。

我在谷歌度过了一个美好的暑假！
