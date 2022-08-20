***

## 标题：“V8中的地图（隐藏类）”&#xA;描述： “V8 如何跟踪和优化对象的感知结构？

让我们展示 V8 如何构建它的隐藏类。主要数据结构包括：

*   `Map`：隐藏类本身。它是对象中的第一个指针值，因此可以轻松比较两个对象是否具有相同的类。
*   `DescriptorArray`：此类具有的属性的完整列表以及有关这些属性的信息。在某些情况下，属性值甚至在此数组中。
*   `TransitionArray`：来自此的“边缘”数组`Map`到兄弟地图。每个边都是一个属性名称，应将其视为“如果我要将具有此名称的属性添加到当前类，我将转换到哪个类？

因为很多`Map`对象只有一个过渡到另一个过渡（即，它们是“过渡”映射，只在通往其他东西的路上使用），V8并不总是创建一个完整的`TransitionArray`为它。相反，它只是直接链接到这个“下一个”`Map`.系统必须在`DescriptorArray`的`Map`被指向，以便找出附加到过渡的名称。

这是一个非常丰富的主题。但是，如果您了解本文中的概念，那么将来的更改也应该可以逐步理解。

## 为什么有隐藏的类？

当然，V8 可以不带隐藏类。它将每个对象视为一个属性包。然而，一个非常有用的原则将被搁置：智能设计的原则。V8 推测您只能创建这么多**不同**对象的种类。每一种对象都将以最终可以被看作是刻板的方式使用。我说“最终被看到”是因为JavaScript语言是一种脚本语言，而不是预编译的语言。所以V8永远不知道接下来会发生什么。为了利用智能设计（即假设代码背后有一个思想），V8必须观察和等待，让结构感渗透进来。隐藏类机制是执行此操作的主要方法。当然，它以复杂的监听机制为前提，这些是已经写了很多关于内联缓存（IC）的内容。

所以，如果你确信这是好的和必要的工作，请跟随我！

## 示例

```javascript
function Peak(name, height, extra) {
  this.name = name;
  this.height = height;
  if (isNaN(extra)) {
    this.experience = extra;
  } else {
    this.prominence = extra;
  }
}

m1 = new Peak("Matterhorn", 4478, 1040);
m2 = new Peak("Wendelstein", 1838, "good");
```

有了这个代码，我们已经从根映射（也称为初始映射）中获得了一个有趣的映射树，该映射附加到函数`Peak`:

<figure>
  <img src="/_img/docs/hidden-classes/drawing-one.svg" width="400" height="480" alt="Hidden class example" loading="lazy">
</figure>

每个蓝色框都是一个地图，从初始地图开始。这是返回的对象的映射，如果以某种方式，我们设法运行了该函数`Peak`而不添加单个属性。后续映射是通过添加由映射之间边缘上的名称给出的属性而生成的映射。每个地图都有一个与该地图的对象关联的属性列表。此外，它还描述了每个属性的确切位置。最后，从这些地图之一，比如说，`Map3`这是对象的隐藏类，如果您为`extra`参数`Peak()`，您可以按照备份链接一直到初始地图。

让我们用这个额外的信息再次绘制它。注释 （i0）、（i1） 表示对象内字段位置 0、1 等：

<figure>
  <img src="/_img/docs/hidden-classes/drawing-two.svg" width="400" height="480" alt="Hidden class example" loading="lazy">
</figure>

现在，如果您在创建至少7个地图之前花时间检查这些地图`Peak`对象，你会遇到**松弛跟踪**这可能会令人困惑。我有[另一篇文章](https://v8.dev/blog/slack-tracking)关于这一点。只需再创建7个对象，它就会完成。此时，您的 Peak 对象将正好具有 3 个对象内属性，无法更直接地在对象中添加。任何其他属性都将卸载到对象的属性支持存储。它只是一个属性值数组，其索引来自映射（从技术上讲，来自`DescriptorArray`附加到地图）。让我们将属性添加到`m2`在新行上，然后再次查看地图树：

```javascript
m2.cost = "one arm, one leg";
```

<figure>
  <img src="/_img/docs/hidden-classes/drawing-three.svg" width="400" height="480" alt="Hidden class example" loading="lazy">
</figure>

我把东西偷偷放在这里。请注意，所有属性都用“const”注释，这意味着从 V8 的角度来看，自构造函数以来，没有人更改过它们，因此一旦初始化，它们就可以被视为常量。TurboFan（优化编译器）喜欢这个。说`m2`被函数引用为常量全局变量。然后查找`m2.cost`可以在编译时完成，因为该字段被标记为常量。我将在本文后面再次讨论这一点。

请注意，属性“成本”标记为`const p0`，这意味着它是存储在索引零处的常量属性**属性后备存储**而不是直接在对象中。这是因为我们在对象中没有更多的空间。此信息可见于`%DebugPrint(m2)`:

    d8> %DebugPrint(m2);
    DebugPrint: 0x2f9488e9: [JS_OBJECT_TYPE]
     - map: 0x219473fd <Map(HOLEY_ELEMENTS)> [FastProperties]
     - prototype: 0x2f94876d <Object map = 0x21947335>
     - elements: 0x419421a1 <FixedArray[0]> [HOLEY_ELEMENTS]
     - properties: 0x2f94aecd <PropertyArray[3]> {
        0x419446f9: [String] in ReadOnlySpace: #name: 0x237125e1
            <String[11]: #Wendelstein> (const data field 0)
        0x23712581: [String] in OldSpace: #height:
            1838 (const data field 1)
        0x23712865: [String] in OldSpace: #experience: 0x237125f9
            <String[4]: #good> (const data field 2)
        0x23714515: [String] in OldSpace: #cost: 0x23714525
            <String[16]: #one arm, one leg>
            (const data field 3) properties[0]
     }
     ...
    {name: "Wendelstein", height: 1, experience: "good", cost: "one arm, one leg"}
    d8>

您可以看到我们有 4 个属性，全部标记为 const。对象中的前 3 个，以及对象中的最后一个`properties[0]`这意味着属性后备存储的第一个槽。我们可以看看：

    d8> %DebugPrintPtr(0x2f94aecd)
    DebugPrint: 0x2f94aecd: [PropertyArray]
     - map: 0x41942be9 <Map>
     - length: 3
     - hash: 0
             0: 0x23714525 <String[16]: #one arm, one leg>
           1-2: 0x41942329 <undefined>

额外的属性就在那里，以防万一您突然决定添加更多属性。

## 真实结构

在这一点上，我们可以做不同的事情，但是由于您必须非常喜欢V8，因此在阅读了本文之后，我想尝试绘制我们使用的真实数据结构，即在`Map`,`DescriptorArray`和`TransitionArray`.既然您已经对幕后构建的隐藏类概念有了一些了解，那么您不妨通过正确的名称和结构将您的思维与代码更紧密地绑定在一起。让我尝试在 V8 的表示中重现最后一个数字。首先，我要画**描述符数组**，其中包含给定地图的属性列表。这些数组是可以共享的 - 关键是Map本身知道在DescriptorArray中允许查看多少属性。由于属性按其添加时间顺序排列，因此这些数组可由多个映射共享。看：

<figure>
  <img src="/_img/docs/hidden-classes/drawing-four.svg" width="600" height="480" alt="Hidden class example" loading="lazy">
</figure>

请注意，**地图1**,**地图2**和**地图3**全部指向**描述符数组1**.每个 Map 中“描述符”字段旁边的数字指示描述符数组中属于 Map 的字段数。所以**地图1**，它只知道“name”属性，只查看中列出的第一个属性**描述符数组1**.而**地图2**有两个属性，“名称”和“高度”。因此，它查看了中的第一项和第二项**描述符数组1**（姓名和身高）。这种共享可以节省大量空间。

当然，我们不能在有分裂的地方分享。如果添加了“体验”属性，则从 Map2 过渡到 Map4;如果添加了“突出”属性，则从 Map3 过渡到 Map3。您可以看到Map4和Map4共享DescriptorArray2，就像DescriptorArray1在三个Map之间共享一样。

我们的“逼真”图中唯一缺少的是`TransitionArray`在这一点上，这仍然是隐喻性的。让我们改变这一点。我冒昧地删除了**后指针**线条，这稍微清理了一下。请记住，从树中的任何地图中，您也可以走上树。

<figure>
  <img src="/_img/docs/hidden-classes/drawing-five.svg" width="600" height="480" alt="Hidden class example" loading="lazy">
</figure>

图表奖励学习。**问题：如果在“名称”之后添加新的属性“评级”，而不是继续使用“高度”和其他属性，会发生什么情况？**

**答**：Map1会得到一个真正的**过渡阵列**以便跟踪分叉。如果属性*高度*已添加，我们应该过渡到**地图2**.但是，如果属性*额定值*添加后，我们就应该去一张新地图，**地图6**.这张地图需要一个新的描述符数组，其中提到*名字*和*额定值*.此时，对象在对象中具有额外的可用插槽（仅使用三个插槽中的一个），因此属性*额定值*将获得其中一个插槽。

*我在的帮助下检查了我的答案`%DebugPrintPtr()`，并绘制以下内容：*

<figure>
  <img src="/_img/docs/hidden-classes/drawing-six.svg" width="500" height="480" alt="Hidden class example" loading="lazy">
</figure>

不用乞求我停下来，我看到这是这种图表的上限！但我认为你可以了解零件是如何移动的。想象一下，如果在添加此替代属性后*额定值*，我们继续*高度*,*经验*和*成本*.好吧，我们必须创建地图**地图7**,**地图8**和**地图9**.因为我们坚持在已建立的地图链中间添加此属性，因此我们将复制许多结构。我没有心思画那幅画——不过如果你把它寄给我，我会把它添加到这份文件中，:)。

我用了方便[梦之堡](https://dreampuf.github.io/GraphvizOnline)项目以轻松制作图表。这是一个[链接](https://dreampuf.github.io/GraphvizOnline/#digraph%20G%20%7B%0A%0A%20%20node%20%5Bfontname%3DHelvetica%2C%20shape%3D%22record%22%2C%20fontcolor%3D%22white%22%2C%20style%3Dfilled%2C%20color%3D%22%233F53FF%22%5D%0A%20%20edge%20%5Bfontname%3DHelvetica%5D%0A%20%20%0A%20%20Map0%20%5Blabel%3D%22%7B%3Ch%3E%20Map0%20%7C%20%3Cd%3E%20descriptors%20\(0\)%20%7C%20%3Ct%3E%20transitions%20\(1\)%7D%22%5D%3B%0A%20%20Map1%20%5Blabel%3D%22%7B%3Ch%3E%20Map1%20%7C%20%3Cd%3E%20descriptors%20\(1\)%20%7C%20%3Ct%3E%20transitions%20\(1\)%7D%22%5D%3B%0A%20%20Map2%20%5Blabel%3D%22%7B%3Ch%3E%20Map2%20%7C%20%3Cd%3E%20descriptors%20\(2\)%20%7C%20%3Ct%3E%20transitions%20\(2\)%7D%22%5D%3B%0A%20%20Map3%20%5Blabel%3D%22%7B%3Ch%3E%20Map3%20%7C%20%3Cd%3E%20descriptors%20\(3\)%20%7C%20%3Ct%3E%20transitions%20\(0\)%7D%22%5D%0A%20%20Map4%20%5Blabel%3D%22%7B%3Ch%3E%20Map4%20%7C%20%3Cd%3E%20descriptors%20\(3\)%20%7C%20%3Ct%3E%20transitions%20\(1\)%7D%22%5D%0A%20%20Map5%20%5Blabel%3D%22%7B%3Ch%3E%20Map5%20%7C%20%3Cd%3E%20descriptors%20\(4\)%20%7C%20%3Ct%3E%20transitions%20\(0\)%7D%22%5D%0A%20%20Map6%20%5Blabel%3D%22%7B%3Ch%3E%20Map6%20%7C%20%3Cd%3E%20descriptors%20\(2\)%20%7C%20%3Ct%3E%20transitions%20\(0\)%7D%22%5D%3B%0A%20%20Map0%3At%20-%3E%20Map1%20%5Blabel%3D%22name%20\(inferred\)%22%5D%3B%0A%20%20%0A%20%20Map4%3At%20-%3E%20Map5%20%5Blabel%3D%22cost%20\(inferred\)%22%5D%3B%0A%20%20%0A%20%20%2F%2F%20Create%20the%20descriptor%20arrays%0A%20%20node%20%5Bfontname%3DHelvetica%2C%20shape%3D%22record%22%2C%20fontcolor%3D%22black%22%2C%20style%3Dfilled%2C%20color%3D%22%23FFB34D%22%5D%3B%0A%20%20%0A%20%20DA0%20%5Blabel%3D%22%7BDescriptorArray0%20%7C%20\(empty\)%7D%22%5D%0A%20%20Map0%3Ad%20-%3E%20DA0%3B%0A%20%20DA1%20%5Blabel%3D%22%7BDescriptorArray1%20%7C%20name%20\(const%20i0\)%20%7C%20height%20\(const%20i1\)%20%7C%20prominence%20\(const%20i2\)%7D%22%5D%3B%0A%20%20Map1%3Ad%20-%3E%20DA1%3B%0A%20%20Map2%3Ad%20-%3E%20DA1%3B%0A%20%20Map3%3Ad%20-%3E%20DA1%3B%0A%20%20%0A%20%20DA2%20%5Blabel%3D%22%7BDescriptorArray2%20%7C%20name%20\(const%20i0\)%20%7C%20height%20\(const%20i1\)%20%7C%20experience%20\(const%20i2\)%20%7C%20cost%20\(const%20p0\)%7D%22%5D%3B%0A%20%20Map4%3Ad%20-%3E%20DA2%3B%0A%20%20Map5%3Ad%20-%3E%20DA2%3B%0A%20%20%0A%20%20DA3%20%5Blabel%3D%22%7BDescriptorArray3%20%7C%20name%20\(const%20i0\)%20%7C%20rating%20\(const%20i1\)%7D%22%5D%3B%0A%20%20Map6%3Ad%20-%3E%20DA3%3B%0A%20%20%0A%20%20%2F%2F%20Create%20the%20transition%20arrays%0A%20%20node%20%5Bfontname%3DHelvetica%2C%20shape%3D%22record%22%2C%20fontcolor%3D%22white%22%2C%20style%3Dfilled%2C%20color%3D%22%23B3813E%22%5D%3B%0A%20%20TA0%20%5Blabel%3D%22%7BTransitionArray0%20%7C%20%3Ca%3E%20experience%20%7C%20%3Cb%3E%20prominence%7D%22%5D%3B%0A%20%20Map2%3At%20-%3E%20TA0%3B%0A%20%20TA0%3Aa%20-%3E%20Map4%3Ah%3B%0A%20%20TA0%3Ab%20-%3E%20Map3%3Ah%3B%0A%20%20%0A%20%20TA1%20%5Blabel%3D%22%7BTransitionArray1%20%7C%20%3Ca%3E%20rating%20%7C%20%3Cb%3E%20height%7D%22%5D%3B%0A%20%20Map1%3At%20-%3E%20TA1%3B%0A%20%20TA1%3Ab%20-%3E%20Map2%3B%0A%20%20TA1%3Aa%20-%3E%20Map6%3B%0A%7D)到上图。

## 涡轮风扇和恒定性能

到目前为止，所有这些字段都标记在`DescriptorArray`如`const`.让我们玩这个。在调试版本上运行以下代码：

```javascript
// run as:
// d8 --allow-natives-syntax --no-lazy-feedback-allocation --code-comments --print-opt-code
function Peak(name, height) {
  this.name = name;
  this.height = height;
}

let m1 = new Peak("Matterhorn", 4478);
m2 = new Peak("Wendelstein", 1838);

// Make sure slack tracking finishes.
for (let i = 0; i < 7; i++) new Peak("blah", i);

m2.cost = "one arm, one leg";
function foo(a) {
  return m2.cost;
}

foo(3);
foo(3);
%OptimizeFunctionOnNextCall(foo);
foo(3);
```

您将获得优化功能的打印输出`foo()`.代码非常短。您将在函数的末尾看到：

    ...
    40  mov eax,0x2a812499          ;; object: 0x2a812499 <String[16]: #one arm, one leg>
    45  mov esp,ebp
    47  pop ebp
    48  ret 0x8                     ;; return "one arm, one leg"!

TurboFan，作为一个厚脸皮的魔鬼，只是直接插入了价值`m2.cost`.好吧，你喜欢这样！

当然，在最后一次调用之后`foo()`你可以插入这一行：

```javascript
m2.cost = "priceless";
```

您认为会发生什么？有一件事是肯定的，我们不能让`foo()`保持原样。它会返回错误的答案。重新运行程序，但添加标志`--trace-deopt`因此，当从系统中删除优化的代码时，您将被告知。打印输出后优化`foo()`，您将看到以下行：

    [marking dependent code 0x5c684901 0x21e525b9 <SharedFunctionInfo foo> (opt #0) for deoptimization,
        reason: field-const]
    [deoptimize marked code in all contexts]

哇。

<figure>
  <img src="/_img/docs/hidden-classes/i_like_it_a_lot.gif" width="440" height="374" alt="I like it a lot" loading="lazy">
</figure>

如果你强制重新优化，你会得到的代码不是很好，但仍然从我们一直在描述的Map结构中受益匪浅。请记住，从我们的图表中，该属性*成本*是
对象的属性支持存储。好吧，它可能已经失去了它的const名称，但我们仍然有它的地址。基本上，在具有映射的对象中**地图5**，我们肯定会验证全局变量`m2`还是有，我们只需要——

1.  加载属性后备存储，以及
2.  读出第一个数组元素。

让我们看看。在最后一行下方添加以下代码：

```javascript
// Force reoptimization of foo().
foo(3);
%OptimizeFunctionOnNextCall(foo);
foo(3);
```

现在看一下生成的代码：

    ...
    40  mov ecx,0x42cc8901          ;; object: 0x42cc8901 <Peak map = 0x3d5873ad>
    45  mov ecx,[ecx+0x3]           ;; Load the properties backing store
    48  mov eax,[ecx+0x7]           ;; Get the first element.
    4b  mov esp,ebp
    4d  pop ebp
    4e  ret 0x8                     ;; return it in register eax!

哎呀。这正是我们所说的应该发生的事情。也许我们开始知道了。

TurboFan也足够智能，可以在可变时进行去优化`m2`永远更改为其他类。您可以看到最新的优化代码再次使用类似内容的droll进行去优化：

```javascript
m2 = 42;  // heh.
```

## 从这里到哪里去

许多选项。映射迁移。字典模式（又名“慢速模式”）。在这个领域有很多值得探索的地方，我希望你和我一样喜欢自己 - 感谢您的阅读！
