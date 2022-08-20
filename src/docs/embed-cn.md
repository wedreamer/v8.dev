***

## 标题： “嵌入 V8 入门”&#xA;描述： “本文档介绍了一些关键的 V8 概念，并提供了一个”hello world“示例，帮助您开始使用 V8 代码。

本文档介绍了一些关键的 V8 概念，并提供了一个“hello world”示例，帮助您开始使用 V8 代码。

## 观众

本文档适用于希望将 V8 JavaScript 引擎嵌入到C++应用程序中的C++程序员。它可以帮助您使自己的应用程序C++对象和方法可用于 JavaScript，并使 JavaScript 对象和函数可用于C++应用程序。

## 世界您好

让我们看一下[Hello World 示例](https://chromium.googlesource.com/v8/v8/+/branch-heads/6.8/samples/hello-world.cc)将 JavaScript 语句作为字符串参数，将其作为 JavaScript 代码执行，并将结果打印到标准输出。

首先，一些关键概念：

*   隔离是具有自己的堆的 VM 实例。
*   本地句柄是指向对象的指针。所有 V8 对象都使用句柄进行访问。由于 V8 垃圾回收器的工作方式，它们是必要的。
*   句柄作用域可以被视为任意数量句柄的容器。完成句柄后，无需单独删除每个句柄，只需删除其作用域即可。
*   上下文是一个执行环境，它允许在 V8 的单个实例中运行单独的、不相关的 JavaScript 代码。您必须显式指定要在其中运行任何 JavaScript 代码的上下文。

这些概念将在[高级指南](/docs/embed#advanced-guide).

## 运行示例

请按照以下步骤自行运行该示例：

1.  通过以下方式下载 V8 源代码[Git 指令](/docs/source-code#using-git).

2.  此 hello world 示例的说明上次已使用 V8 v7.1.11 进行了测试。您可以查看此分支`git checkout refs/tags/7.1.11 -b sample -t`

3.  使用帮助程序脚本创建生成配置：

    ```bash
    tools/dev/v8gen.py x64.release.sample
    ```

    您可以通过运行以下命令来检查和手动编辑生成配置：

    ```bash
    gn args out.gn/x64.release.sample
    ```

4.  在 Linux 64 系统上构建静态库：

    ```bash
    ninja -C out.gn/x64.release.sample v8_monolith
    ```

5.  编译`hello-world.cc`，链接到在构建过程中创建的静态库。例如，在使用 GNU 编译器的 64 位 Linux 上：

    ```bash
    g++ -I. -Iinclude samples/hello-world.cc -o hello_world -lv8_monolith -Lout.gn/x64.release.sample/obj/ -pthread -std=c++14 -DV8_COMPRESS_POINTERS
    ```

6.  对于更复杂的代码，V8 在没有 ICU 数据文件的情况下失败。将此文件复制到存储二进制文件的位置：

    ```bash
    cp out.gn/x64.release.sample/icudtl.dat .
    ```

7.  运行`hello_world`命令行中的可执行文件。例如，在 Linux 上，在 V8 目录中，运行：

    ```bash
    ./hello_world
    ```

8.  它打印`Hello, World!`.耶！

如果您正在寻找与master同步的示例，请查看该文件[`hello-world.cc`](https://chromium.googlesource.com/v8/v8/+/master/samples/hello-world.cc).这是一个非常简单的示例，您可能希望做的不仅仅是将脚本作为字符串执行。[下面的高级指南](#advanced-guide)包含有关 V8 嵌入器的更多信息。

## 更多示例代码

以下示例作为源代码下载的一部分提供。

### [`process.cc`](https://github.com/v8/v8/blob/master/samples/process.cc)

此示例提供了扩展假设的 HTTP 请求处理应用程序（例如，该应用程序可能是 Web 服务器的一部分）所需的代码，以便它可以编写脚本。它采用 JavaScript 脚本作为参数，该参数必须提供一个名为`Process`.The JavaScript`Process`函数可用于，例如，收集诸如虚构Web服务器提供的每个页面获得的点击量之类的信息。

### [`shell.cc`](https://github.com/v8/v8/blob/master/samples/shell.cc)

此示例将文件名作为参数，然后读取并执行其内容。包括一个命令提示符，您可以在其中输入 JavaScript 代码段，然后执行这些代码段。在此示例中，其他函数如`print`也通过使用对象和函数模板添加到 JavaScript 中。

## 高级指南

现在，您已经熟悉了将 V8 用作独立虚拟机以及一些关键的 V8 概念（如句柄、作用域和上下文），现在让我们进一步讨论这些概念，并介绍一些其他概念，这些概念对于在您自己的 C++ 应用程序中嵌入 V8 至关重要。

V8 API 提供用于编译和执行脚本、访问C++方法和数据结构、处理错误以及启用安全检查的函数。您的应用程序可以使用 V8，就像任何其他C++库一样。您的C++代码通过包含标头通过 V8 API 访问 V8`include/v8.h`.

### 句柄和垃圾回收

句柄提供对 JavaScript 对象在堆中的位置的引用。V8 垃圾回收器回收无法再访问的对象所使用的内存。在垃圾回收过程中，垃圾回收器通常会将对象移动到堆中的不同位置。当垃圾回收器移动对象时，垃圾回收器还会使用对象的新位置更新引用该对象的所有句柄。

如果一个对象无法从 JavaScript 访问，并且没有引用它的句柄，则该对象被视为垃圾。垃圾回收器会不时删除所有被视为垃圾的对象。V8 的垃圾回收机制是 V8 性能的关键。

有几种类型的句柄：

*   本地句柄保存在堆栈上，并在调用相应的析构函数时被删除。这些句柄的生存期由句柄作用域确定，句柄作用域通常在函数调用开始时创建。删除句柄作用域时，垃圾回收器可以自由地解除分配以前由句柄作用域中的句柄引用的那些对象，前提是它们不再可从 JavaScript 或其他句柄访问。这种类型的句柄在上面的 hello world 示例中使用。

    本地句柄具有类`Local<SomeType>`.

    **注意：**句柄堆栈不是C++调用堆栈的一部分，但句柄作用域嵌入在C++堆栈中。句柄作用域只能是堆栈分配的，不能与`new`.

*   持久句柄提供对堆分配的 JavaScript 对象的引用，就像本地句柄一样。有两种类型，它们在它们处理的引用的生存期管理方面有所不同。如果需要为多个函数调用保留对对象的引用，或者当句柄生存期与C++作用域不对应时，请使用持久句柄。例如，Google Chrome使用持久句柄来引用文档对象模型（DOM）节点。持久的句柄可以变得很弱，使用`PersistentBase::SetWeak`，以在对对象的唯一引用来自弱持久性句柄时触发来自垃圾回收器的回调。

    *   一个`UniquePersistent<SomeType>`handle 依赖于C++构造函数和析构函数来管理基础对象的生存期。
    *   一个`Persistent<SomeType>`可以使用其构造函数构造，但必须使用显式清除`Persistent::Reset`.

*   还有其他类型的句柄很少使用，我们在这里只简要提及：

    *   `Eternal`是预期永远不会被删除的 JavaScript 对象的持久句柄。它使用起来更便宜，因为它使垃圾回收器免于确定该对象的活动性。
    *   双`Persistent`和`UniquePersistent`无法复制，这使得它们不适合用作预C++11标准库容器的值。`PersistentValueMap`和`PersistentValueVector`为持久值提供容器类，具有映射和类似向量的语义。C++11嵌入器不需要这些，因为C++11移动语义解决了潜在的问题。

当然，每次创建对象时创建本地句柄都会导致很多句柄！这就是句柄作用域非常有用的地方。您可以将句柄作用域视为包含大量句柄的容器。当调用句柄作用域的析构函数时，将从堆栈中删除在该作用域内创建的所有句柄。如您所料，这会导致垃圾回收器有资格从堆中删除句柄点的对象。

返回[我们非常简单的你好世界示例](#hello-world)，在下图中，您可以看到句柄堆栈和堆分配的对象。请注意，`Context::New()`返回`Local`句柄，我们创建一个新的`Persistent`基于它的句柄来演示`Persistent`处理。

![](/\_img/docs/embed/local-persist-handles-review.png)

当析构函数`HandleScope::~HandleScope`，则删除句柄作用域。如果已删除句柄作用域中的句柄引用的对象没有其他引用，则这些对象在下一个垃圾回收中有资格被删除。垃圾回收器还可以删除`source_obj`和`script_obj`堆中的对象，因为它们不再被任何句柄引用或以其他方式可以从 JavaScript 访问。由于上下文句柄是持久句柄，因此在退出句柄作用域时不会将其删除。 删除上下文句柄的唯一方法是显式调用`Reset`在上面。

：：：备注
**注意：**在本文档中，术语“句柄”是指本地句柄。在讨论持久句柄时，该术语被完整地使用。
:::

重要的是要意识到此模型的一个常见陷阱：*不能直接从声明句柄作用域的函数返回本地句柄*.如果执行尝试返回的本地句柄，则在函数返回之前，句柄作用域的析构函数最终会将其删除。返回本地句柄的正确方法是构造一个`EscapableHandleScope`而不是`HandleScope`并调用`Escape`方法，传入要返回其值的句柄。以下是在实践中如何工作的示例：

```cpp
// This function returns a new array with three elements, x, y, and z.
Local<Array> NewPointArray(int x, int y, int z) {
  v8::Isolate* isolate = v8::Isolate::GetCurrent();

  // We will be creating temporary handles so we use a handle scope.
  v8::EscapableHandleScope handle_scope(isolate);

  // Create a new empty array.
  v8::Local<v8::Array> array = v8::Array::New(isolate, 3);

  // Return an empty result if there was an error creating the array.
  if (array.IsEmpty())
    return v8::Local<v8::Array>();

  // Fill out the values
  array->Set(0, Integer::New(isolate, x));
  array->Set(1, Integer::New(isolate, y));
  array->Set(2, Integer::New(isolate, z));

  // Return the value through Escape.
  return handle_scope.Escape(array);
}
```

这`Escape`方法将其参数的值复制到封闭的作用域中，删除其所有本地句柄，然后返回可以安全返回的新句柄副本。

### 上下文

在 V8 中，上下文是一个执行环境，它允许单独的、不相关的 JavaScript 应用程序在 V8 的单个实例中运行。您必须显式指定要在其中运行任何 JavaScript 代码的上下文。

为什么这是必要的？因为 JavaScript 提供了一组内置的实用程序函数和对象，这些函数和对象可以通过 JavaScript 代码进行更改。例如，如果两个完全不相关的JavaScript函数都以相同的方式更改了全局对象，那么意外的结果很可能会发生。

就 CPU 时间和内存而言，考虑到必须构建的内置对象的数量，创建新的执行上下文似乎是一项成本高昂的操作。但是，V8 的广泛缓存确保了，虽然您创建的第一个上下文有些昂贵，但后续上下文要便宜得多。这是因为第一个上下文需要创建内置对象并解析内置的 JavaScript 代码，而后续上下文只需要为其上下文创建内置对象。使用 V8 快照功能（使用构建选项激活）`snapshot=yes`，这是默认设置）创建第一个上下文所花费的时间将得到高度优化，因为快照包括一个序列化的堆，其中包含已编译的内置 JavaScript 代码代码的代码。除了垃圾回收之外，V8 的广泛缓存也是 V8 性能的关键。

创建上下文后，可以任意次数地进入和退出它。当您处于上下文 A 中时，还可以输入不同的上下文 B，这意味着您将 A 替换为 B 作为当前上下文。退出 B 时，A 将恢复为当前上下文。如下图所示：

![](/\_img/docs/embed/intro-contexts.png)

请注意，每个上下文的内置实用程序函数和对象保持独立。您可以选择在创建上下文时设置安全令牌。查看[安全模型](#security-model)部分以获取更多信息。

在 V8 中使用上下文的动机是，浏览器中的每个窗口和 iframe 都可以有自己全新的 JavaScript 环境。

### 模板

模板是上下文中 JavaScript 函数和对象的蓝图。您可以使用模板将C++函数和数据结构包装在 JavaScript 对象中，以便 JavaScript 脚本可以操作它们。例如，Google Chrome 使用模板将C++ DOM 节点包装为 JavaScript 对象，并在全局命名空间中安装函数。您可以创建一组模板，然后对创建的每个新上下文使用相同的模板。您可以根据需要拥有任意数量的模板。但是，在任何给定上下文中，您只能有任何模板的一个实例。

在JavaScript中，函数和对象之间存在很强的二元性。要在 Java 或 C++中创建新类型的对象，通常需要定义一个新类。在 JavaScript 中，您可以改为创建一个新函数，并使用该函数作为构造函数创建实例。JavaScript 对象的布局和功能与构造它的函数密切相关。这反映在 V8 模板的工作方式中。有两种类型的模板：

*   函数模板

    函数模板是单个函数的蓝图。您可以通过调用模板的`GetFunction`方法，从您希望在其中实例化 JavaScript 函数的上下文中。您还可以将C++回调与函数模板相关联，该模板在调用 JavaScript 函数实例时调用。

*   对象模板

    每个函数模板都有一个关联的对象模板。这用于配置使用此函数作为其构造函数创建的对象。您可以将两种类型的C++回调与对象模板相关联：

    *   当脚本访问特定对象属性时，将调用访问器回调
    *   当脚本访问任何对象属性时，将调用侦听器回调

    [访问](#accessors)和[拦截 器](#interceptors)本文档稍后将对此进行讨论。

下面的代码提供了为全局对象创建模板并设置内置全局函数的示例。

```cpp
// Create a template for the global object and set the
// built-in global functions.
v8::Local<v8::ObjectTemplate> global = v8::ObjectTemplate::New(isolate);
global->Set(v8::String::NewFromUtf8(isolate, "log"),
            v8::FunctionTemplate::New(isolate, LogCallback));

// Each processor gets its own context so different processors
// do not affect each other.
v8::Persistent<v8::Context> context =
    v8::Context::New(isolate, nullptr, global);
```

此示例代码取自`JsHttpProcessor::Initializer`在`process.cc`样本。

### 访问

访问器是一个C++回调，它在 JavaScript 脚本访问对象属性时计算并返回值。访问器通过对象模板进行配置，使用`SetAccessor`方法。此方法采用与其关联的属性的名称，并在脚本尝试读取或写入属性时运行两个回调。

访问器的复杂性取决于您正在操作的数据类型：

*   [访问静态全局变量](#accessing-static-global-variables)
*   [访问动态变量](#accessing-dynamic-variables)

### 访问静态全局变量

假设有两个C++整数变量，`x`和`y`它们将作为上下文中的全局变量提供给 JavaScript。为此，每当脚本读取或写入C++访问器函数时，都需要调用这些函数。这些访问器函数 C++使用`Integer::New`，并使用 将 JavaScript 整数转换为C++整数`Int32Value`.下面提供了一个示例：

```cpp
void XGetter(v8::Local<v8::String> property,
              const v8::PropertyCallbackInfo<Value>& info) {
  info.GetReturnValue().Set(x);
}

void XSetter(v8::Local<v8::String> property, v8::Local<v8::Value> value,
             const v8::PropertyCallbackInfo<void>& info) {
  x = value->Int32Value();
}

// YGetter/YSetter are so similar they are omitted for brevity

v8::Local<v8::ObjectTemplate> global_templ = v8::ObjectTemplate::New(isolate);
global_templ->SetAccessor(v8::String::NewFromUtf8(isolate, "x"),
                          XGetter, XSetter);
global_templ->SetAccessor(v8::String::NewFromUtf8(isolate, "y"),
                          YGetter, YSetter);
v8::Persistent<v8::Context> context =
    v8::Context::v8::New(isolate, nullptr, global_templ);
```

请注意，上述代码中的对象模板是与上下文同时创建的。该模板可以提前创建，然后用于任意数量的上下文。

### 访问动态变量

在前面的示例中，变量是静态和全局的。如果正在操作的数据是动态的，就像浏览器中的 DOM 树一样，该怎么办？让我们想象一下`x`和`y`是C++类上的对象字段`Point`:

```cpp
class Point {
 public:
  Point(int x, int y) : x_(x), y_(y) { }
  int x_, y_;
}
```

进行任意数量的C++`point`对于可用于 JavaScript 的实例，我们需要为每个C++创建一个 JavaScript 对象。`point`并在 JavaScript 对象和C++实例之间建立连接。这是使用外部值和内部对象字段完成的。

首先为`point`包装对象：

```cpp
v8::Local<v8::ObjectTemplate> point_templ = v8::ObjectTemplate::New(isolate);
```

每个 JavaScript`point`对象保留对C++对象的引用，它是具有内部字段的包装器。之所以这样命名，是因为无法从 JavaScript 中访问它们，只能从C++代码访问它们。一个对象可以有任意数量的内部字段，内部字段的数量在对象模板上设置如下：

```cpp
point_templ->SetInternalFieldCount(1);
```

此处，内部字段计数设置为`1`这意味着该对象有一个内部字段，索引为`0`，指向C++对象。

添加`x`和`y`模板的访问器：

```cpp
point_templ->SetAccessor(v8::String::NewFromUtf8(isolate, "x"),
                         GetPointX, SetPointX);
point_templ->SetAccessor(v8::String::NewFromUtf8(isolate, "y"),
                         GetPointY, SetPointY);
```

接下来，通过创建模板的新实例，然后设置内部字段来包装C++点`0`到点周围的外部包装`p`.

```cpp
Point* p = ...;
v8::Local<v8::Object> obj = point_templ->NewInstance();
obj->SetInternalField(0, v8::External::New(isolate, p));
```

外部对象只是围绕`void*`.外部对象只能用于在内部字段中存储引用值。JavaScript 对象不能直接引用C++对象，因此外部值被用作从 JavaScript 到C++的“桥梁”。 从这个意义上说，外部值与句柄相反，因为句柄允许C++引用JavaScript对象。

这是`get`和`set`的访问器`x`这`y`访问器定义是相同的，除了`y`取代`x`:

```cpp
void GetPointX(Local<String> property,
               const PropertyCallbackInfo<Value>& info) {
  v8::Local<v8::Object> self = info.Holder();
  v8::Local<v8::External> wrap =
      v8::Local<v8::External>::Cast(self->GetInternalField(0));
  void* ptr = wrap->Value();
  int value = static_cast<Point*>(ptr)->x_;
  info.GetReturnValue().Set(value);
}

void SetPointX(v8::Local<v8::String> property, v8::Local<v8::Value> value,
               const v8::PropertyCallbackInfo<void>& info) {
  v8::Local<v8::Object> self = info.Holder();
  v8::Local<v8::External> wrap =
      v8::Local<v8::External>::Cast(self->GetInternalField(0));
  void* ptr = wrap->Value();
  static_cast<Point*>(ptr)->x_ = value->Int32Value();
}
```

访问器提取对`point`对象，该对象由 JavaScript 对象包装，然后读取并写入关联的字段。这样，这些泛型访问器就可以用于任意数量的包装点对象。

### 拦截 器

还可以为脚本访问任何对象属性时指定回调。这些称为拦截器。为了提高效率，有两种类型的拦截器：

*   *命名属性拦截器*- 在访问具有字符串名称的属性时调用。
    在浏览器环境中，这方面的一个例子是`document.theFormName.elementName`.
*   *索引属性拦截器*- 访问索引属性时调用。在浏览器环境中，这方面的一个例子是`document.forms.elements[0]`.

示例`process.cc`，随 V8 源代码一起提供，包括一个使用拦截器的示例。在以下代码片段中`SetNamedPropertyHandler`指定`MapGet`和`MapSet`拦截 器：

```cpp
v8::Local<v8::ObjectTemplate> result = v8::ObjectTemplate::New(isolate);
result->SetNamedPropertyHandler(MapGet, MapSet);
```

这`MapGet`下面提供了拦截器：

```cpp
void JsHttpRequestProcessor::MapGet(v8::Local<v8::String> name,
                                    const v8::PropertyCallbackInfo<Value>& info) {
  // Fetch the map wrapped by this object.
  map<string, string> *obj = UnwrapMap(info.Holder());

  // Convert the JavaScript string to a std::string.
  string key = ObjectToString(name);

  // Look up the value if it exists using the standard STL idiom.
  map<string, string>::iterator iter = obj->find(key);

  // If the key is not present return an empty handle as signal.
  if (iter == obj->end()) return;

  // Otherwise fetch the value and wrap it in a JavaScript string.
  const string &value = (*iter).second;
  info.GetReturnValue().Set(v8::String::NewFromUtf8(
      value.c_str(), v8::String::kNormalString, value.length()));
}
```

与访问器一样，只要访问属性，就会调用指定的回调。访问器和拦截器之间的区别在于，拦截器处理所有属性，而访问器与一个特定属性相关联。

### 安全模型

“同源策略”（首先在 Netscape Navigator 2.0 中引入）可防止从一个“源”加载的文档或脚本从不同的“源”获取或设置文档的属性。术语 origin 在这里被定义为域名的组合（例如`www.example.com`）、协议（例如`https`） 和端口。例如`www.example.com:81`与原点不同`www.example.com`.所有这三个网页必须匹配，两个网页才能被视为具有相同的来源。如果没有此保护，恶意网页可能会损害另一个网页的完整性。

在 V8 中，“原点”被定义为上下文。默认情况下，不允许访问除要从中调用的上下文以外的任何上下文。若要访问从中调用的上下文以外的上下文，需要使用安全令牌或安全回调。安全令牌可以是任何值，但通常是一个符号，一个在其他任何地方都不存在的规范字符串。您可以选择使用以下命令指定安全令牌`SetSecurityToken`设置上下文时。如果未指定安全令牌，V8 将为您创建的上下文自动生成一个安全令牌。

当尝试访问全局变量时，V8 安全系统首先根据尝试访问全局对象的代码的安全令牌检查正在访问的全局对象的安全令牌。如果令牌匹配，则授予访问权限。如果令牌不匹配，V8 将执行回调以检查是否应允许访问。您可以通过在对象上设置安全回调来指定是否应允许访问对象，使用`SetAccessCheckCallbacks`方法在对象模板上。然后，V8 安全系统可以获取被访问对象的安全回调，并调用它以询问是否允许另一个上下文访问它。此回调为正在访问的对象、正在访问的属性的名称、访问类型（例如读取、写入或删除）提供，并返回是否允许访问。

此机制在 Google Chrome 中实现，因此，如果安全令牌不匹配，则使用特殊回调仅允许访问以下内容：`window.focus()`,`window.blur()`,`window.close()`,`window.location`,`window.open()`,`history.forward()`,`history.back()`和`history.go()`.

### 异常

如果发生错误，V8 将引发异常 — 例如，当脚本或函数尝试读取不存在的属性时，或者如果调用的函数不是函数。

如果操作未成功，V8 将返回空句柄。因此，在继续执行之前，代码必须检查返回值是否为空句柄，这一点很重要。使用`Local`类的公共成员函数`IsEmpty()`.

您可以使用以下命令捕获异常`TryCatch`例如：

```cpp
v8::TryCatch trycatch(isolate);
v8::Local<v8::Value> v = script->Run();
if (v.IsEmpty()) {
  v8::Local<v8::Value> exception = trycatch.Exception();
  v8::String::Utf8Value exception_str(exception);
  printf("Exception: %s\n", *exception_str);
  // ...
}
```

如果返回的值是空句柄，并且您没有`TryCatch`到位后，您的代码必须得到救助。如果您确实有`TryCatch`捕获异常，并允许您的代码继续处理。

### 遗产

JavaScript 是一个*无类*，面向对象的语言，因此，它使用原型继承而不是经典继承。对于接受过传统面向对象语言（如C++和Java）培训的程序员来说，这可能会令人费解。

基于类的面向对象语言（如 Java 和 C++）基于两个不同实体的概念：类和实例。JavaScript是一种基于原型的语言，因此没有这种区别：它只是有对象。JavaScript本身不支持类层次结构的声明;但是，JavaScript的原型机制简化了将自定义属性和方法添加到对象的所有实例的过程。在 JavaScript 中，您可以向对象添加自定义属性。例如：

```js
// Create an object named `bicycle`.
function bicycle() {}
// Create an instance of `bicycle` called `roadbike`.
var roadbike = new bicycle();
// Define a custom property, `wheels`, on `roadbike`.
roadbike.wheels = 2;
```

以这种方式添加的自定义属性仅存在于该对象的实例中。如果我们创建另一个实例`bicycle()`叫`mountainbike`例如`mountainbike.wheels`会再来的`undefined`除非`wheels`属性是显式添加的。

有时这正是必需的，在其他时候，将自定义属性添加到对象的所有实例中会很有帮助 - 毕竟所有自行车都有轮子。这就是JavaScript的原型对象非常有用的地方。若要使用原型对象，请引用关键字`prototype`在将自定义属性添加到对象之前，如下所示：

```js
// First, create the “bicycle” object
function bicycle() {}
// Assign the wheels property to the object’s prototype
bicycle.prototype.wheels = 2;
```

的所有实例`bicycle()`现在将具有`wheels`属性预内置到它们中。

V8 中对模板使用相同的方法。每`FunctionTemplate`具有`PrototypeTemplate`方法，它为函数的原型提供模板。C++ 您可以在`PrototypeTemplate`然后，它将出现在相应实例的所有实例上`FunctionTemplate`.例如：

```cpp
v8::Local<v8::FunctionTemplate> biketemplate = v8::FunctionTemplate::New(isolate);
biketemplate->PrototypeTemplate().Set(
    v8::String::NewFromUtf8(isolate, "wheels"),
    v8::FunctionTemplate::New(isolate, MyWheelsMethodCallback)->GetFunction()
);
```

这会导致所有实例`biketemplate`拥有`wheels`其原型链中的方法，当调用时，会导致C++函数`MyWheelsMethodCallback`被调用。

V8的`FunctionTemplate`类提供公共成员函数`Inherit()`当您希望函数模板从另一个函数模板继承时，可以调用它，如下所示：

```cpp
void Inherit(v8::Local<v8::FunctionTemplate> parent);
```
