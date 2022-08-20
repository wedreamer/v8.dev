***

标题： 'V8 发布 v8.0'
作者：“Leszek Swirski，他的名字的V8th”
化身：

*   'leszek-swirski'
    日期： 2019-12-18
    标签：
*   释放
    描述： 'V8 v8.0 具有可选的链式链接、空合并、更快的高阶内置功能 — 哦，由于指针压缩，内存使用量减少了 40%，没什么大不了的。
    推文：“1207323849861279746”

***

<!-- Yes, it's an SVG. Please don't ask me how long I spent making it. -->

<!-- markdownlint-capture -->

<!-- markdownlint-disable no-inline-html -->

<div style="position: relative; left: 50%; margin-left: -45vw; width: 90vw; pointer-events: none">
<div style="width: 1075px; max-width:100%; margin:0 auto">
<svg xmlns="http://www.w3.org/2000/svg" width="1075" height="260" viewBox="-5 140 1075 260" style="display:block;width:100%;height:auto;margin-top:-4%;margin-bottom:-1em"><style>a{pointer-events: auto}text{font-family:Helvetica,Roboto,Segoe UI,Calibri,sans-serif;fill:#1c2022;font-weight:400}.bg,.divider{stroke:#e1e8ed;stroke-width:.8;fill:#fff}a.name text{font-weight:700}.subText,a.like text,a.name .subText{fill:#697882;font-size:14px;font-weight:400}a.like path{fill:url(#b)}a:hover text,a:focus text{fill:#3b94d9}a.like:hover text,a.like:focus text{fill:#e0245e}a.like:hover path,a.like:focus path{fill:url(#B)}.dark .bg{stroke:#66757f;fill:#000}.dark text{fill:#f5f8fa}.dark .subText,.dark a.name .subText,.dark a.like text{fill:#8899a6}.dark a:hover text,.dark a:focus text{fill:#55acee}.dark a.like:hover text,.dark a.like:focus text{fill:#e0245e}</style><defs><pattern id="a" width="1" height="1" patternContentUnits="objectBoundingBox" patternUnits="objectBoundingBox"><image width="1" height="1" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 72 72%22><path fill=%22none%22 d=%22M0 0h72v72H0z%22/><path class=%22icon%22 fill=%22%231da1f2%22 d=%22M68.812 15.14c-2.348 1.04-4.87 1.744-7.52 2.06 2.704-1.62 4.78-4.186 5.757-7.243-2.53 1.5-5.33 2.592-8.314 3.176C56.35 10.59 52.948 9 49.182 9c-7.23 0-13.092 5.86-13.092 13.093 0 1.026.118 2.02.338 2.98C25.543 24.527 15.9 19.318 9.44 11.396c-1.125 1.936-1.77 4.184-1.77 6.58 0 4.543 2.312 8.552 5.824 10.9-2.146-.07-4.165-.658-5.93-1.64-.002.056-.002.11-.002.163 0 6.345 4.513 11.638 10.504 12.84-1.1.298-2.256.457-3.45.457-.845 0-1.666-.078-2.464-.23 1.667 5.2 6.5 8.985 12.23 9.09-4.482 3.51-10.13 5.605-16.26 5.605-1.055 0-2.096-.06-3.122-.184 5.794 3.717 12.676 5.882 20.067 5.882 24.083 0 37.25-19.95 37.25-37.25 0-.565-.013-1.133-.038-1.693 2.558-1.847 4.778-4.15 6.532-6.774z%22/></svg>"/></pattern><pattern id="b" width="1" height="1" patternContentUnits="objectBoundingBox" patternUnits="objectBoundingBox"><image width="1" height="1" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 width=%2224%22 height=%2224%22 viewBox=%220 0 24 24%22><path class=%22icon%22 fill=%22%23697882%22 d=%22M12 21.638h-.014C9.403 21.59 1.95 14.856 1.95 8.478c0-3.064 2.525-5.754 5.403-5.754 2.29 0 3.83 1.58 4.646 2.73.813-1.148 2.353-2.73 4.644-2.73 2.88 0 5.404 2.69 5.404 5.755 0 6.375-7.454 13.11-10.037 13.156H12zM7.354 4.225c-2.08 0-3.903 1.988-3.903 4.255 0 5.74 7.035 11.596 8.55 11.658 1.52-.062 8.55-5.917 8.55-11.658 0-2.267-1.822-4.255-3.902-4.255-2.528 0-3.94 2.936-3.952 2.965-.23.562-1.156.562-1.387 0-.015-.03-1.426-2.965-3.955-2.965z%22/></svg>"/></pattern><pattern id="B" width="1" height="1" patternContentUnits="objectBoundingBox" patternUnits="objectBoundingBox"><image width="1" height="1" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 width=%2224%22 height=%2224%22 viewBox=%220 0 24 24%22><path class=%22icon%22 fill=%22%23E0245E%22 d=%22M12 21.638h-.014C9.403 21.59 1.95 14.856 1.95 8.478c0-3.064 2.525-5.754 5.403-5.754 2.29 0 3.83 1.58 4.646 2.73.813-1.148 2.353-2.73 4.644-2.73 2.88 0 5.404 2.69 5.404 5.755 0 6.375-7.454 13.11-10.037 13.156H12zM7.354 4.225c-2.08 0-3.903 1.988-3.903 4.255 0 5.74 7.035 11.596 8.55 11.658 1.52-.062 8.55-5.917 8.55-11.658 0-2.267-1.822-4.255-3.902-4.255-2.528 0-3.94 2.936-3.952 2.965-.23.562-1.156.562-1.387 0-.015-.03-1.426-2.965-3.955-2.965z%22/></svg>"/></pattern></defs><g><path class="bg" d="M-2.2 222.4l398.4-34.8 13.6 127.3-398.4 34.8z"/><g transform="rotate(-5 830.8 -212.3) scale(.8)"><image width="36" height="36" x="-25.2" y="206.2" href="/_img/v8-release-80/twitter-avatar-1.jpg"/><a class="name"><text x="66" y="21"><tspan x="19.8" y="218.6">Josebaba 💥</tspan></text><text x="66" y="42" class="subText"><tspan x="19.8" y="235.4">@fullstackmofo</tspan></text></a><path fill="url(#a)" d="M412.8 206.2h20v20h-20z"/><text x="21" y="72" class="subText"><tspan x="-25.2" y="266.8">Replying to @v8js</tspan></text></a><text x="21" y="93"><tspan x="-25.2" y="291.2">V8 almost at v8</tspan></text><a class="like"><path d="M-25.2 307.4h17.5v17.5h-17.5z"/><text x="42" y="125" class="subText"><tspan x="-4.7" y="321.2">4</tspan></text></a><a href="https://twitter.com/fullstackmofo/status/1197260632237780994"><text x="61" y="126" class="subText"><tspan x="15.1" y="321.2">22:09 - 20 Nov 2019</tspan></text></a></g></g><g><path class="bg" d="M147.2 238.9l399 27.9-10.8 127-399-28z"/><g transform="rotate(4 -638.7 1274.7) scale(.8)"><image width="36" height="36" x="112.3" y="254.2" href="/_img/v8-release-80/twitter-avatar-2.jpg"/><a class="name"><text x="66" y="21"><tspan x="157.3" y="264.8">Connor ‘Stryxus’ Shearer</tspan></text><text x="66" y="40" class="subText"><tspan x="157.3" y="281.6">@Stryxus</tspan></text></a><path fill="url(#a)" d="M550.3 254.2h20v20h-20z"/><text x="21" y="71" class="subText"><tspan x="112.3" y="314">Replying to @v8js</tspan></text><g data-id="p"><text x="21" y="92"><tspan x="112.3" y="339.2">What happens when v8 reaches v8? 🤔</tspan></text></g><a class="like"><path d="M112.3 355.4h17.5v17.5h-17.5z"/><text x="42" y="125"><tspan x="132.8" y="369.2">11</tspan></text></a><a  href="https://twitter.com/Stryxus/status/1197187677747122176"><text x="68" y="126" class="subText"><tspan x="159.4" y="369.2">17:19 - 20 Nov 2019</tspan></text></a></g></g><g><path class="bg" d="M383.2 179.6l399.8-14 5.4 126.6-399.8 14z"/><g transform="rotate(-2 1958.9 -3131) scale(.8)"><image width="36" height="36" x="356.8" y="174.2" href="/_img/v8-release-80/twitter-avatar-3.jpg"/><a class="name"><text x="66" y="21"><tspan x="401.8" y="184.8">Thibault Molleman</tspan></text><text x="66" y="40" class="subText"><tspan x="401.8" y="201.6">@thibaultmol</tspan></text></a><path fill="url(#a)" d="M794.8 174.2h20v20h-20z"/><text x="21" y="71" class="subText"><tspan x="356.8" y="234">Replying to @v8js</tspan></text><text x="21" y="92"><tspan x="356.8" y="258.4">Wait. What happens when we get V8 V8?</tspan></text><a class="like"><path d="M356.8 274.6h17.5v17.5h-17.5z"/></a><a href="https://twitter.com/thibaultmol/status/1141656354169470976"><text x="54" y="125" class="subText"><tspan x="389.3" y="288.4">11:37 - 20 Jun 2019</tspan></text></a></g></g><g><path class="bg" d="M522 272.1l400-7 2.6 127.4-400 7z"/><g transform="rotate(-1 4619.2 -7976.5) scale(.8)"><image width="36" height="36" x="494.3" y="270.2" href="/_img/v8-release-80/twitter-avatar-4.jpg"/><a class="name"><text x="66" y="21"><tspan x="539.3" y="280.8">Greg Miernicki</tspan></text><text x="66" y="40" class="subText"><tspan x="539.3" y="297.6">@gregulatore</tspan></text></a><path fill="url(#a)" d="M932.3 270.2h20v20h-20z"/><text x="21" y="71" class="subText"><tspan x="494.3" y="330">Replying to @v8js</tspan></text><g data-id="p"><text x="21" y="92"><tspan x="494.3" y="355.2">Anything special planned for v8 v8.0? 😅</tspan></text></g><a class="like"><path d="M494.3 371.4h17.5v17.5h-17.5z"/><text x="42" y="125"><tspan x="514.8" y="385.2">5</tspan></text></a><a href="https://twitter.com/gregulatore/status/1161302336314191872"><text x="61" y="126" class="subText"><tspan x="534.6" y="385.2">16:43 - 13 Aug 2019</tspan></text></a></g></g><g><path class="bg" d="M671.2 141.3l394 69.5-30 142.7-394-69.5z"/><g transform="rotate(10 469.6 1210) scale(.8)"><image width="36" height="36" x="624.2" y="174.2" href="/_img/v8-release-80/twitter-avatar-5.jpg"/><a class="name"><text x="66" y="21"><tspan x="669.2" y="184.8">SignpostMarv</tspan></text><text x="66" y="40" class="subText"><tspan x="669.2" y="201.6">@SignpostMarv</tspan></text></a><path fill="url(#a)" d="M1062.2 174.2h20v20h-20z"/><text x="21" y="71" class="subText"><tspan x="624.2" y="234">Replying to @v8js @ChromiumDev</tspan></text><text x="21" y="92"><tspan x="624.2" y="258.4">are you going to be having an extra special party when V8 goes</tspan><tspan x="624.2" y="279.4">v8?</tspan></text><a class="like"><path d="M624.2 296.6h17.5v17.5h-17.5z"/><text x="42" y="146"><tspan x="644.7" y="310.4">18</tspan></text></a><a href="https://twitter.com/SignpostMarv/status/1177603910288203782"><text x="69" y="147" class="subText"><tspan x="672.3" y="310.4">16:20 - 27 Sep 2019</tspan></text></a></g></g></svg>
</div>
</div>
<!-- markdownlint-restore -->

终于来了。每个 V8 版本，每六周一次，当我们作为我们的一部分进行分支时[发布流程](/docs/release-process)，问题来了，当 V8 达到版本 8 时会发生什么。我们会有派对吗？我们会发布一个新的编译器吗？我们会跳过版本8和9，只停留在永恒的V8版本X吗？最后，在之后[10年以上](/blog/10-years)在我们的第100篇博客文章中，我们很高兴地宣布我们最新的分支机构，[V8 ~~版本 8.0~~ V8](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/8.0)，我们终于可以回答这个问题了：

这是错误修复和性能改进。

这篇文章提供了一些亮点的预览，预计几周后将与Chrome 80 Stable协调发布。

## 性能（大小和速度）{ #performance }

### 指针压缩

\~~我们改变了我们所有的`void *`自`pv`，将源文件大小减少多达 66%。~~

V8 堆包含一整组项目，例如浮点值、字符串字符、编译代码和标记值（表示进入 V8 堆的指针或小整数）。在检查堆时，我们发现这些标记的值占据了堆的大部分！

标记的值与系统指针一样大：对于 32 位体系结构，它们的宽度为 32 位，对于 64 位体系结构，它们为 64 位宽。然后，在比较 32 位版本和 64 位版本时，我们对每个标记值使用的堆内存是其两倍。

对我们来说幸运的是，我们有一个诀窍。顶部位可以从较低位合成。然后，我们只需要将唯一的较低位存储到堆中，从而节省宝贵的内存资源...平均节省40%的堆内存！

![Pointer compression saves an average of 40% of memory.](/\_img/v8-release-80/pointer-compression-chart.svg)

在改善内存时，通常以牺牲性能为代价。通常。我们很自豪地宣布，在V8中花费的时间内，我们看到真实网站的性能有所提高，并且垃圾收集器也是如此！

：：：表包装器
|                      ||桌面|移动|
|-------------|----------|---------|--------|
|脸书|V8-总|-8% |-6% |
|^^          |指导性案例|-10% |-17% |
|美国有线电视新闻网|V8-总|-3% |-8% |
|^^          |指导性案例|-14% |-20% |
|谷歌地图|V8-总|-4% |-6% |
|^^          |指导性案例|-7% |-12% |
:::

如果指针压缩激起了您的兴趣，请留意包含更多详细信息的完整博客文章。

### 优化高阶内置

我们最近删除了TurboFan优化管道中的一个限制，该限制阻止了对高阶内置组件的主动优化。

```js
const charCodeAt = Function.prototype.call.bind(String.prototype.charCodeAt);

charCodeAt(string, 8);
```

到目前为止，呼吁`charCodeAt`对于 TurboFan 来说是完全不透明的，这导致了对用户定义函数的通用调用的生成。通过此更改，我们现在能够认识到我们实际上是在调用内置`String.prototype.charCodeAt`函数，因此能够触发TurboFan库存中的所有进一步优化，以改善对内置的调用，从而获得与以下相同的性能：

```js
string.charCodeAt(8);
```

此更改会影响许多其他内置功能，例如`Function.prototype.apply`,`Reflect.apply`，以及许多高阶数组内置（例如`Array.prototype.map`).

## JavaScript

### 可选链接

在编写属性访问链时，程序员通常需要检查中间值是否为空（即`null`或`undefined`).没有错误检查的链可能会抛出，而具有显式错误检查的链是冗长的，并且具有检查所有真实值而不是仅检查非空值的不需要的结果。

```js
// Error prone-version, could throw.
const nameLength = db.user.name.length;

// Less error-prone, but harder to read.
let nameLength;
if (db && db.user && db.user.name)
  nameLength = db.user.name.length;
```

[可选链接](https://v8.dev/features/optional-chaining)(`?.`） 允许程序员编写更简洁、更健壮的属性访问链，以检查中间值是否为空。如果中间值为 null，则整个表达式的计算结果为`undefined`.

```js
// Still checks for errors and is much more readable.
const nameLength = db?.user?.name?.length;
```

除了静态属性访问外，还支持动态属性访问和调用。请参阅我们的[功能说明](https://v8.dev/features/optional-chaining)有关详细信息和更多示例。

### 空合并

这[空合并](https://v8.dev/features/nullish-coalescing)算子`??`是一个新的短路二元运算符，用于处理默认值。目前，默认值有时使用逻辑处理`||`运算符，如下面的示例所示。

```js
function Component(props) {
  const enable = props.enabled || true;
  // …
}
```

用途`||`不适合计算默认值，因为`a || b`评估为`b`什么时候`a`是虚假的。如果`props.enabled`已显式设置为`false`,`enable`仍然是真的。

使用空合并运算符，`a ?? b`评估为`b`什么时候`a`为空 （`null`或`undefined`），否则计算为`a`.这是所需的默认值行为，并使用`??`修复了上面的错误。

```js
function Component(props) {
  const enable = props.enabled ?? true;
  // …
}
```

空合并运算符和可选链接是配套功能，可以很好地协同工作。该示例可以进一步修改，以处理没有`props`参数被传入。

```js
function Component(props) {
  const enable = props?.enabled ?? true;
  // …
}
```

请参阅我们的[功能说明](https://v8.dev/features/nullish-coalescing)有关详细信息和更多示例。

## V8 接口

请使用`git log branch-heads/7.9..branch-heads/8.0 include/v8.h`以获取 API 更改的列表。

具有[活动 V8 结账](/docs/source-code#using-git)可以使用`git checkout -b 8.0 -t branch-heads/8.0`以试验 V8 v8.0 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
