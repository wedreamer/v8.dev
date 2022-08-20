***

标题： 'V8 版本 v9.4'
作者： '英格瓦·斯捷潘扬 （[@RReverser](https://twitter.com/RReverser))'
化身：

*   '英格瓦-斯捷潘扬'
    日期： 2021-09-06
    标签：
*   释放
    描述： 'V8 版本 v9.4 将类静态初始化块引入 JavaScript。
    推文：“1434915404418277381”

***

每六周，我们创建一个新的 V8 分支，作为我们[发布流程](https://v8.dev/docs/release-process).每个版本都是在Chrome Beta里程碑之前从V8的Git主版本分支出来的。今天，我们很高兴地宣布我们最新的分行，[V8 版本 9.4](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/9.4)，直到几周后与Chrome 94 Stable合作发布之前，它一直处于测试阶段。V8 v9.4 充满了各种面向开发人员的好东西。这篇文章提供了对发布预期的一些亮点的预览。

## JavaScript

### 类静态初始化块

类能够通过静态初始化块对每个类评估应运行一次的代码进行分组。

```javascript
class C {
  // This block will run when the class itself is evaluated
  static { console.log("C's static block"); }
}
```

从 v9.4 开始，类静态初始化块将可用，而无需`--harmony-class-static-blocks`旗。有关这些块的作用域的所有详细语义，请参阅[我们的解释员](https://v8.dev/features/class-static-initializer-blocks).

## V8 接口

请使用`git log branch-heads/9.3..branch-heads/9.4 include/v8.h`以获取 API 更改的列表。

具有有效 V8 签出功能的开发人员可以使用`git checkout -b 9.4 -t branch-heads/9.4`以试验 V8 v9.4 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
