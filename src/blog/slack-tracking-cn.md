***

标题： “V8中的松弛跟踪”
作者： '迈克尔·斯坦顿 （[@alpencoder](https://twitter.com/alpencoder)），著名大师*松弛*'
描述： “详细了解 V8 松弛跟踪机制。
化身：

*   “迈克尔-斯坦顿”
    日期： 2020-09-24 14：00：00
    标签：
*   内部

***

松弛跟踪是一种为新对象提供初始大小的方法，该大小为**大于他们可能实际使用的数量**，以便他们可以快速添加新属性。然后，经过一段时间后，到**神奇地将未使用的空间返回到系统**.很整洁，对吧？

它特别有用，因为JavaScript没有静态类。系统永远无法“一目了然地”看到您有多少属性。引擎一个接一个地体验它们。所以当你读到：

```js
function Peak(name, height) {
  this.name = name;
  this.height = height;
}

const m1 = new Peak('Matterhorn', 4478);
```

你可能会认为引擎拥有它运行良好所需的一切 - 毕竟，你已经告诉它对象有两个属性。但是，V8真的不知道接下来会发生什么。此对象`m1`可以传递给另一个函数，该函数向其添加 10 个以上的属性。Slack跟踪源于这种需要对环境中接下来的任何事情做出响应，而无需静态编译来推断整体结构。它就像 V8 中的许多其他机制一样，其基础只是你通常可以说的关于执行的事情，比如：

*   大多数物体很快就会死亡，很少有物体能长寿——垃圾回收“代际假说”。
*   该计划确实有一个组织结构 - 我们建立[形状或“隐藏类”](https://mathiasbynens.be/notes/shapes-ics)（我们称之为**地图**在V8）中，我们看到程序员使用的对象，因为我们相信它们会很有用。*顺便说一句，[V8中的快速属性](/blog/fast-properties)是一个很棒的帖子，有关于地图和物业访问的有趣细节。*
*   程序具有初始化状态，当一切都是新的并且很难说出什么是重要的时。之后，可以通过稳定的使用来识别重要的类和函数 - 我们的反馈制度和编译器管道就是从这个想法中发展出来的。

最后，也是最重要的一点，运行时环境必须非常快，否则我们只是在哲学化。

现在，V8 可以简单地将属性存储在附加到主对象的后备存储中。与直接存在于对象中的属性不同，此后备存储可以通过复制和替换指针来无限增长。但是，对属性的最快访问是通过避免该间接寻址并从对象的开头查看固定偏移量来实现的。下面，我展示了 V8 堆中具有两个对象内属性的普通 JavaScript 对象的布局。前三个词在每个对象中都是标准的（指向映射、属性支持存储和元素支持存储的指针）。您可以看到该对象无法“增长”，因为它很难与堆中的下一个对象相对应：

![](../_img/slack-tracking/property-layout.svg)

：：：备注
**注意：**我省略了物业后备商店的细节，因为目前唯一重要的是它可以随时用更大的商店替换。但是，它也是 V8 堆上的一个对象，并且像驻留在其中的所有对象一样具有映射指针。
:::

所以无论如何，由于对象内属性提供的性能，V8愿意在每个对象中给你额外的空间，并且**松弛跟踪**是它完成的方式。最终，你会安定下来，停止添加新的房产，并开始挖掘比特币或其他业务。

V8给了你多少“时间”？巧妙地，它考虑了您构造特定对象的次数。事实上，地图中有一个计数器，它用系统中一个更神秘的魔术数字初始化：**七**.

另一个问题：V8如何知道在物体体中提供多少额外空间？它实际上从编译过程获得提示，该过程提供了估计数量的属性。此计算包括来自原型对象的属性数，以递归方式沿原型链向上排列。最后，为了更好地衡量，它增加了**八**更多（另一个神奇的数字！您可以在`JSFunction::CalculateExpectedNofProperties()`:

```cpp
int JSFunction::CalculateExpectedNofProperties(Isolate* isolate,
                                               Handle<JSFunction> function) {
  int expected_nof_properties = 0;
  for (PrototypeIterator iter(isolate, function, kStartAtReceiver);
       !iter.IsAtEnd(); iter.Advance()) {
    Handle<JSReceiver> current =
        PrototypeIterator::GetCurrent<JSReceiver>(iter);
    if (!current->IsJSFunction()) break;
    Handle<JSFunction> func = Handle<JSFunction>::cast(current);

    // The super constructor should be compiled for the number of expected
    // properties to be available.
    Handle<SharedFunctionInfo> shared(func->shared(), isolate);
    IsCompiledScope is_compiled_scope(shared->is_compiled_scope(isolate));
    if (is_compiled_scope.is_compiled() ||
        Compiler::Compile(func, Compiler::CLEAR_EXCEPTION,
                          &is_compiled_scope)) {
      DCHECK(shared->is_compiled());
      int count = shared->expected_nof_properties();
      // Check that the estimate is sensible.
      if (expected_nof_properties <= JSObject::kMaxInObjectProperties - count) {
        expected_nof_properties += count;
      } else {
        return JSObject::kMaxInObjectProperties;
      }
    } else {
      // In case there was a compilation error proceed iterating in case there
      // will be a builtin function in the prototype chain that requires
      // certain number of in-object properties.
      continue;
    }
  }
  // In-object slack tracking will reclaim redundant inobject space
  // later, so we can afford to adjust the estimate generously,
  // meaning we over-allocate by at least 8 slots in the beginning.
  if (expected_nof_properties > 0) {
    expected_nof_properties += 8;
    if (expected_nof_properties > JSObject::kMaxInObjectProperties) {
      expected_nof_properties = JSObject::kMaxInObjectProperties;
    }
  }
  return expected_nof_properties;
}
```

让我们来看看我们的对象`m1`从之前：

```js
function Peak(name, height) {
  this.name = name;
  this.height = height;
}

const m1 = new Peak('Matterhorn', 4478);
```

通过计算`JSFunction::CalculateExpectedNofProperties`和我们的`Peak()`函数，我们应该有2个对象属性，并且由于松弛跟踪，另外8个额外的属性。我们可以打印`m1`跟`%DebugPrint()`(*这个方便的函数公开了地图结构。您可以通过运行以下命令来使用它`d8`带旗`--allow-natives-syntax`*):

    > %DebugPrint(m1);
    DebugPrint: 0x49fc866d: [JS_OBJECT_TYPE]
     - map: 0x58647385 <Map(HOLEY_ELEMENTS)> [FastProperties]
     - prototype: 0x49fc85e9 <Object map = 0x58647335>
     - elements: 0x28c821a1 <FixedArray[0]> [HOLEY_ELEMENTS]
     - properties: 0x28c821a1 <FixedArray[0]> {
        0x28c846f9: [String] in ReadOnlySpace: #name: 0x5e412439 <String[10]: #Matterhorn> (const data field 0)
        0x5e412415: [String] in OldSpace: #height: 4478 (const data field 1)
     }
      0x58647385: [Map]
     - type: JS_OBJECT_TYPE
     - instance size: 52
     - inobject properties: 10
     - elements kind: HOLEY_ELEMENTS
     - unused property fields: 8
     - enum length: invalid
     - stable_map
     - back pointer: 0x5864735d <Map(HOLEY_ELEMENTS)>
     - prototype_validity cell: 0x5e4126fd <Cell value= 0>
     - instance descriptors (own) #2: 0x49fc8701 <DescriptorArray[2]>
     - prototype: 0x49fc85e9 <Object map = 0x58647335>
     - constructor: 0x5e4125ed <JSFunction Peak (sfi = 0x5e4124dd)>
     - dependent code: 0x28c8212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
     - construction counter: 6

请注意，对象的实例大小为 52。V8 中的对象布局如下所示：

|单词 |什么|
|---- |---------------------------------------------------- |
|0 |地图|
|1 |指向属性数组|的指针
|2 |指向元素数组|的指针
|3 |对象内字段 1（指向字符串的指针）`"Matterhorn"`) |
|4 |对象内字段 2（整数值`4478`)             |
|5 |未使用的对象内字段 3 |
|...    |...                                                    |
|12 |未使用的对象字段 10 |

在这个32位二进制文件中，指针大小为4，因此我们得到了每个普通JavaScript对象都有的3个初始单词，然后在对象中增加了10个单词。它在上面告诉我们，有8个“未使用的属性字段”。因此，我们正在经历松弛跟踪。我们的对象是臃肿，贪婪的珍贵字节的消费者！

我们如何瘦身？我们在地图中使用施工计数器字段。我们达到零，然后决定我们已经完成了松弛跟踪。但是，如果构造更多对象，则不会看到上面的计数器减少。为什么？

好吧，这是因为上面显示的地图不是“该”地图`Peak`对象。它只是从**初始地图**该`Peak`对象是在执行构造函数代码之前给出的。

如何找到初始地图？令人高兴的是，功能`Peak()`有一个指向它的指针。它是初始映射中的构造计数器，我们用于控制松弛跟踪：

    > %DebugPrint(Peak);
    d8> %DebugPrint(Peak)
    DebugPrint: 0x31c12561: [Function] in OldSpace
     - map: 0x2a2821f5 <Map(HOLEY_ELEMENTS)> [FastProperties]
     - prototype: 0x31c034b5 <JSFunction (sfi = 0x36108421)>
     - elements: 0x28c821a1 <FixedArray[0]> [HOLEY_ELEMENTS]
     - function prototype: 0x37449c89 <Object map = 0x2a287335>
     - initial_map: 0x46f07295 <Map(HOLEY_ELEMENTS)>   // Here's the initial map.
     - shared_info: 0x31c12495 <SharedFunctionInfo Peak>
     - name: 0x31c12405 <String[4]: #Peak>
    …

    d8> // %DebugPrintPtr allows you to print the initial map.
    d8> %DebugPrintPtr(0x46f07295)
    DebugPrint: 0x46f07295: [Map]
     - type: JS_OBJECT_TYPE
     - instance size: 52
     - inobject properties: 10
     - elements kind: HOLEY_ELEMENTS
     - unused property fields: 10
     - enum length: invalid
     - back pointer: 0x28c02329 <undefined>
     - prototype_validity cell: 0x47f0232d <Cell value= 1>
     - instance descriptors (own) #0: 0x28c02135 <DescriptorArray[0]>
     - transitions #1: 0x46f0735d <Map(HOLEY_ELEMENTS)>
         0x28c046f9: [String] in ReadOnlySpace: #name:
             (transition to (const data field, attrs: [WEC]) @ Any) ->
                 0x46f0735d <Map(HOLEY_ELEMENTS)>
     - prototype: 0x5cc09c7d <Object map = 0x46f07335>
     - constructor: 0x21e92561 <JSFunction Peak (sfi = 0x21e92495)>
     - dependent code: 0x28c0212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
     - construction counter: 5

看看施工柜台是如何减少到5的吗？如果您想从我们上面显示的两属性地图中找到初始地图，可以在以下方面遵循其后指针`%DebugPrintPtr()`直到您到达地图`undefined`在后面的指针插槽中。这将是上面的这张地图。

现在，地图树从初始地图中生长出来，每个属性都有一个分支从该点开始添加。我们称之为分支*转换*.在上面的初始地图的打印输出中，您是否看到过渡到带有标签“name”的下一张地图？到目前为止，整个地图树如下所示：

![(X, Y, Z) means (instance size, number of in-object properties, number of unused properties).](../_img/slack-tracking/root-map-1.svg)

这些基于属性名称的转换是[“盲痣”](https://www.google.com/search?q=blind+mole\&tbm=isch)“的 JavaScript 在你身后构建它的映射。此初始映射也存储在函数中`Peak`，因此当它用作构造函数时，该映射可用于设置`this`对象。

```js
const m1 = new Peak('Matterhorn', 4478);
const m2 = new Peak('Mont Blanc', 4810);
const m3 = new Peak('Zinalrothorn', 4221);
const m4 = new Peak('Wendelstein', 1838);
const m5 = new Peak('Zugspitze', 2962);
const m6 = new Peak('Watzmann', 2713);
const m7 = new Peak('Eiger', 3970);
```

这里很酷的事情是，在创建之后`m7`运行`%DebugPrint(m1)`再次产生一个奇妙的新结果：

    DebugPrint: 0x5cd08751: [JS_OBJECT_TYPE]
     - map: 0x4b387385 <Map(HOLEY_ELEMENTS)> [FastProperties]
     - prototype: 0x5cd086cd <Object map = 0x4b387335>
     - elements: 0x586421a1 <FixedArray[0]> [HOLEY_ELEMENTS]
     - properties: 0x586421a1 <FixedArray[0]> {
        0x586446f9: [String] in ReadOnlySpace: #name:
            0x51112439 <String[10]: #Matterhorn> (const data field 0)
        0x51112415: [String] in OldSpace: #height:
            4478 (const data field 1)
     }
    0x4b387385: [Map]
     - type: JS_OBJECT_TYPE
     - instance size: 20
     - inobject properties: 2
     - elements kind: HOLEY_ELEMENTS
     - unused property fields: 0
     - enum length: invalid
     - stable_map
     - back pointer: 0x4b38735d <Map(HOLEY_ELEMENTS)>
     - prototype_validity cell: 0x511128dd <Cell value= 0>
     - instance descriptors (own) #2: 0x5cd087e5 <DescriptorArray[2]>
     - prototype: 0x5cd086cd <Object map = 0x4b387335>
     - constructor: 0x511127cd <JSFunction Peak (sfi = 0x511125f5)>
     - dependent code: 0x5864212d <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
     - construction counter: 0

我们的实例大小现在是 20，即 5 个字：

|单词 |什么|
|---- |------------------------------- |
|0 |地图|
|1 |指向属性数组|的指针
|2 |指向元素数组|的指针
|3 |名称 |
|4 |高度 |

你会想知道这是怎么发生的。毕竟，如果这个对象被放置在内存中，并且曾经有10个属性，那么系统怎么能容忍这8个单词躺在周围，而没有人拥有它们呢？的确，我们从来没有给他们灌输任何有趣的东西——也许这可以帮助我们。

如果你想知道为什么我担心留下这些词，你需要了解一些关于垃圾收集器的背景。对象一个接一个地布局，V8 垃圾回收器通过一遍又一遍地走过它来跟踪内存中的东西。从内存中的第一个单词开始，它期望找到指向地图的指针。它从映射中读取实例大小，然后知道要前进到下一个有效对象的距离。对于某些类，它必须额外计算长度，但这就是它的全部内容。

![](../_img/slack-tracking/gc-heap-1.svg)

在上图中，红色框是**地图**，以及填充对象实例大小的单词的白框。垃圾收集器可以通过从一个地图跳到另一个地图来“行走”堆。

那么，如果地图突然更改其实例大小，会发生什么情况呢？现在，当GC（垃圾回收器）在堆中行走时，它会发现自己在看一个以前没有看到的单词。就我们而言`Peak`类，我们从占用13个单词更改为仅5个（我将“未使用的属性”单词涂成黄色）：

![](../_img/slack-tracking/gc-heap-2.svg)

![](../_img/slack-tracking/gc-heap-3.svg)

如果我们巧妙地用**实例大小为 4 的“填充”映射**.这样，一旦它们暴露在遍历中，GC就会轻轻地走过它们。

![](../_img/slack-tracking/gc-heap-4.svg)

这在 代码中表示`Factory::InitializeJSObjectBody()`:

```cpp
void Factory::InitializeJSObjectBody(Handle<JSObject> obj, Handle<Map> map,
                                     int start_offset) {

  // <lines removed>

  bool in_progress = map->IsInobjectSlackTrackingInProgress();
  Object filler;
  if (in_progress) {
    filler = *one_pointer_filler_map();
  } else {
    filler = *undefined_value();
  }
  obj->InitializeBody(*map, start_offset, *undefined_value(), filler);
  if (in_progress) {
    map->FindRootMap(isolate()).InobjectSlackTrackingStep(isolate());
  }

  // <lines removed>
}
```

这就是行动中的松弛跟踪。对于您创建的每个类，您可以期望它在一段时间内占用更多内存，但在第 7 个实例化时，我们“称之为好”，并公开剩余的空间供 GC 查看。这些一个单词的对象没有所有者 - 也就是说，没有人指向它们 - 所以当集合发生时，它们被释放出来，并且活体对象可能会被压缩以节省空间。

下图反映了松弛跟踪**完成**对于此初始地图。请注意，实例大小现在是 20（5 个单词：映射、属性和元素数组，以及另外 2 个插槽）。松弛跟踪从初始映射开始尊重整个链。也就是说，如果初始映射的后代最终使用了所有 10 个初始额外属性，则初始映射将保留它们，并将它们标记为未使用：

![(X, Y, Z) means (instance size, number of in-object properties, number of unused properties).](../_img/slack-tracking/root-map-2.svg)

现在，slack 跟踪已经完成，如果我们向其中一个属性添加另一个属性会发生什么情况`Peak`对象？

```js
m1.country = 'Switzerland';
```

V8 必须进入属性后备存储。我们最终得到以下对象布局：

|单词 |值|
|---- |------------------------------------- |
|0 |地图|
|1 |指向属性后备存储|的指针
|2 |指向元素（空数组）|
|3 |指向字符串的指针`"Matterhorn"`|
|4 |`4478`|

然后，属性后备存储如下所示：

|单词 |值|
|---- |--------------------------------- |
|0 |地图|
|1 |长度 （3） |
|2 |指向字符串的指针`"Switzerland"`|
|3 |`undefined`|
|4 |`undefined`|
|5 |`undefined`|

我们有这些额外的`undefined`值，以防您决定添加更多属性。我们认为你可能会，根据你到目前为止的行为！

## 可选属性

在某些情况下，您可能会只添加属性。假设如果高度为 4000 米或更高，则要跟踪另外两个属性，`prominence`和`isClimbed`:

```js
function Peak(name, height, prominence, isClimbed) {
  this.name = name;
  this.height = height;
  if (height >= 4000) {
    this.prominence = prominence;
    this.isClimbed = isClimbed;
  }
}
```

您可以添加以下几个不同的变体：

```js
const m1 = new Peak('Wendelstein', 1838);
const m2 = new Peak('Matterhorn', 4478, 1040, true);
const m3 = new Peak('Zugspitze', 2962);
const m4 = new Peak('Mont Blanc', 4810, 4695, true);
const m5 = new Peak('Watzmann', 2713);
const m6 = new Peak('Zinalrothorn', 4221, 490, true);
const m7 = new Peak('Eiger', 3970);
```

在本例中，对象`m1`,`m3`,`m5`和`m7`有一个地图和对象`m2`,`m4`和`m6`由于附加属性，在初始映射的后代链中进一步向下放置一个映射。完成此地图族的松弛追踪后，将有**4**对象内属性，而不是**2**与之前一样，因为松弛跟踪可确保为初始地图下方地图树中的任何后代使用的最大对象内属性保留足够的空间。

下面显示了运行上述代码后的地图系列，当然，slack跟踪已完成：

![(X, Y, Z) means (instance size, number of in-object properties, number of unused properties).](../_img/slack-tracking/root-map-3.svg)

## 优化代码怎么样？

让我们在 slack 跟踪完成之前编译一些优化的代码。我们将使用几个本机语法命令来强制在完成松弛跟踪之前进行优化编译：

```js
function foo(a1, a2, a3, a4) {
  return new Peak(a1, a2, a3, a4);
}

%PrepareFunctionForOptimization(foo);
const m1 = foo('Wendelstein', 1838);
const m2 = foo('Matterhorn', 4478, 1040, true);
%OptimizeFunctionOnNextCall(foo);
foo('Zugspitze', 2962);
```

这应该足以编译和运行优化的代码。我们在TurboFan（优化编译器）中做了一些事情，称为[**创建降低**](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/js-create-lowering.h;l=32;drc=ee9e7e404e5a3f75a3ca0489aaf80490f625ca27)，其中我们内联对象的分配。这意味着我们生成的本机代码会发出指令，要求GC提供要分配的对象的实例大小，然后仔细初始化这些字段。但是，如果 slack 跟踪在稍后的某个时间点停止，则此代码将无效。我们能做些什么呢？

简单易用！我们只是提前结束此地图系列的松弛跟踪。这是有道理的，因为通常情况下，在创建数千个对象之前，我们不会编译优化的函数。所以松弛跟踪*应该*完成。如果不是，那就太糟糕了！无论如何，如果此时创建的对象少于7个，则该对象肯定不是那么重要。（通常，请记住，我们只是在程序运行很长时间后进行优化。

### 在后台线程上编译

我们可以在主线程上编译优化的代码，在这种情况下，我们可以通过一些调用来更改初始映射来过早地结束松弛跟踪，因为世界已经停止。但是，我们会在后台线程上尽可能多地进行编译。从这个线程中，触摸初始地图将是危险的，因为它*可能在运行 JavaScript 的主线程上发生了变化。*所以我们的技术是这样的：

1.  **猜**如果现在确实停止了松弛跟踪，实例大小将会是这样的。记住这个尺寸。
2.  当编译几乎完成时，我们返回到主线程，如果尚未完成，我们可以安全地强制完成松弛跟踪。
3.  检查：实例大小是否符合我们的预测？㞖**我们很好！**如果没有，请丢弃代码对象，稍后再试。

如果你想在代码中看到这一点，看看这个类[`InitialMapInstanceSizePredictionDependency`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/compilation-dependencies.cc?q=InitialMapInstanceSizePredictionDependency\&ss=chromium%2Fchromium%2Fsrc)以及它如何用于`js-create-lowering.cc`以创建内联分配。你会看到`PrepareInstall()`方法在主线程上调用，这将强制完成松弛跟踪。然后方法`Install()`检查我们对实例大小的猜测是否成立。

下面是具有内联分配的优化代码。首先，您看到与GC的通信，检查我们是否可以通过实例大小向前撞击指针并获取该指针（这称为碰撞指针分配）。然后，我们开始填写新对象的字段：

```asm
…
43  mov ecx,[ebx+0x5dfa4]
49  lea edi,[ecx+0x1c]
4c  cmp [ebx+0x5dfa8],edi       ;; hey GC, can we have 28 (0x1c) bytes please?
52  jna 0x36ec4a5a  <+0x11a>

58  lea edi,[ecx+0x1c]
5b  mov [ebx+0x5dfa4],edi       ;; okay GC, we took it. KThxbye.
61  add ecx,0x1                 ;; hells yes. ecx is my new object.
64  mov edi,0x46647295          ;; object: 0x46647295 <Map(HOLEY_ELEMENTS)>
69  mov [ecx-0x1],edi           ;; Store the INITIAL MAP.
6c  mov edi,0x56f821a1          ;; object: 0x56f821a1 <FixedArray[0]>
71  mov [ecx+0x3],edi           ;; Store the PROPERTIES backing store (empty)
74  mov [ecx+0x7],edi           ;; Store the ELEMENTS backing store (empty)
77  mov edi,0x56f82329          ;; object: 0x56f82329 <undefined>
7c  mov [ecx+0xb],edi           ;; in-object property 1 <-- undefined
7f  mov [ecx+0xf],edi           ;; in-object property 2 <-- undefined
82  mov [ecx+0x13],edi          ;; in-object property 3 <-- undefined
85  mov [ecx+0x17],edi          ;; in-object property 4 <-- undefined
88  mov edi,[ebp+0xc]           ;; retrieve argument {a1}
8b  test_w edi,0x1
90  jz 0x36ec4a6d  <+0x12d>
96  mov eax,0x4664735d          ;; object: 0x4664735d <Map(HOLEY_ELEMENTS)>
9b  mov [ecx-0x1],eax           ;; push the map forward
9e  mov [ecx+0xb],edi           ;; name = {a1}
a1  mov eax,[ebp+0x10]          ;; retrieve argument {a2}
a4  test al,0x1
a6  jnz 0x36ec4a77  <+0x137>
ac  mov edx,0x46647385          ;; object: 0x46647385 <Map(HOLEY_ELEMENTS)>
b1  mov [ecx-0x1],edx           ;; push the map forward
b4  mov [ecx+0xf],eax           ;; height = {a2}
b7  cmp eax,0x1f40              ;; is height >= 4000?
bc  jng 0x36ec4a32  <+0xf2>
                  -- B8 start --
                  -- B9 start --
c2  mov edx,[ebp+0x14]          ;; retrieve argument {a3}
c5  test_b dl,0x1
c8  jnz 0x36ec4a81  <+0x141>
ce  mov esi,0x466473ad          ;; object: 0x466473ad <Map(HOLEY_ELEMENTS)>
d3  mov [ecx-0x1],esi           ;; push the map forward
d6  mov [ecx+0x13],edx          ;; prominence = {a3}
d9  mov esi,[ebp+0x18]          ;; retrieve argument {a4}
dc  test_w esi,0x1
e1  jz 0x36ec4a8b  <+0x14b>
e7  mov edi,0x466473d5          ;; object: 0x466473d5 <Map(HOLEY_ELEMENTS)>
ec  mov [ecx-0x1],edi           ;; push the map forward to the leaf map
ef  mov [ecx+0x17],esi          ;; isClimbed = {a4}
                  -- B10 start (deconstruct frame) --
f2  mov eax,ecx                 ;; get ready to return this great Peak object!
…
```

顺便说一句，要看到所有这些，你应该有一个调试版本并传递一些标志。我将代码放入一个文件中，并调用：

```bash
./d8 --allow-natives-syntax --trace-opt --code-comments --print-opt-code mycode.js
```

我希望这是一个有趣的探索。我想特别感谢Igor Sheludko和Maya Armyanova（耐心地！）审查这篇文章。
