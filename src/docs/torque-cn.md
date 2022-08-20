***

## 标题： 'V8扭矩用户手册'&#xA;描述：“本文档解释了 V8 代码库中使用的 V8 扭矩语言。

V8 Torque 是一种语言，它允许为 V8 项目做出贡献的开发人员通过关注*意图*他们对 VM 的更改，而不是专注于不相关的实现细节。该语言设计得足够简单，可以很容易地直接翻译[ECMAScript 规范](https://tc39.es/ecma262/)在 V8 中实现，但功能强大到足以以健壮的方式表达低级 V8 优化技巧，例如基于特定对象形状的测试创建快速路径。

Torque 对于 V8 工程师和 JavaScript 开发人员来说将非常熟悉，它将类似 TypeScript 的语法与语法和类型相结合，该语法和类型简化了 V8 代码的编写和理解。[`CodeStubAssembler`](/blog/csa).凭借强大的类型系统和结构化的控制流程，扭矩确保了结构的正确性。扭矩的表现力足以表达几乎所有的功能[目前在V8的内置组件中找到](/docs/builtin-functions).它也非常可互操作`CodeStubAssembler`内置和`macro`s 以 C++ 编写，允许扭矩代码使用手写的 CSA 功能，反之亦然。

Torque提供了语言结构来表示V8实现的高级，语义丰富的花絮，Torque编译器将这些细节转换为有效的汇编代码`CodeStubAssembler`.Torque的语言结构和Torque编译器的错误检查都确保了正确性，而以前直接使用`CodeStubAssembler`.传统上，编写最佳代码`CodeStubAssembler`要求 V8 工程师在他们的脑海中携带大量专业知识（其中大部分从未在任何书面文档中正式记录过），以避免在实现过程中出现微妙的陷阱。如果没有这些知识，编写高效内置的学习曲线是陡峭的。即使具备必要的知识，非明显和非监管的陷阱也往往会导致正确性或[安全](https://bugs.chromium.org/p/chromium/issues/detail?id=775888) [错误](https://bugs.chromium.org/p/chromium/issues/detail?id=785804).使用Torque，扭矩编译器可以避免和自动识别其中的许多陷阱。

## 开始

在 Torque 中编写的大多数源代码都签入 V8 存储库[这`src/builtins`目录](https://github.com/v8/v8/tree/master/src/builtins)，文件扩展名为`.tq`.V8 堆分配类的扭矩定义与其C++定义一起找到，在`.tq`文件与 中相应的C++文件同名`src/objects`.实际的扭矩编译器可以在[`src/torque`](https://github.com/v8/v8/tree/master/src/torque).扭矩功能测试在[`test/torque`](https://github.com/v8/v8/tree/master/test/torque),[`test/cctest/torque`](https://github.com/v8/v8/tree/master/test/cctest/torque)和[`test/unittests/torque`](https://github.com/v8/v8/tree/master/test/unittests/torque).

为了让您体验一下这门语言，让我们编写一个V8内置的，打印“Hello World！为此，我们将添加一个扭矩`macro`在测试用例中，并从`cctest`测试框架。

首先打开`test/torque/test-torque.tq`文件，并在末尾添加以下代码（但在最后一次关闭之前）`}`):

```torque
@export
macro PrintHelloWorld() {
  Print('Hello world!');
}
```

接下来，打开`test/cctest/torque/test-torque.cc`并添加以下使用新扭矩代码构建代码存根的测试用例：

```cpp
TEST(HelloWorld) {
  Isolate* isolate(CcTest::InitIsolateOnce());
  CodeAssemblerTester asm_tester(isolate, 0);
  TestTorqueAssembler m(asm_tester.state());
  {
    m.PrintHelloWorld();
    m.Return(m.UndefinedConstant());
  }
  FunctionTester ft(asm_tester.GenerateCode(), 0);
  ft.Call();
}
```

然后[构建`cctest`可执行](/docs/test)，最后执行`cctest`测试打印“Hello world”：

```bash
$ out/x64.debug/cctest test-torque/HelloWorld
Hello world!
```

## 扭矩如何生成代码

Torque编译器不直接创建机器代码，而是生成调用V8现有代码C++代码。`CodeStubAssembler`接口。这`CodeStubAssembler`使用[涡轮风扇编译器](https://v8.dev/docs/turbofan)后端生成高效代码。因此，扭矩编译需要多个步骤：

1.  这`gn`build 首先运行 Torque 编译器。它处理所有`*.tq`文件。每个扭矩文件`path/to/file.tq`导致生成以下文件：

    *   `path/to/file-tq-csa.cc`和`path/to/file-tq-csa.h`包含生成的 CSA 宏。
    *   `path/to/file-tq.inc`包含在相应的标头中`path/to/file.h`包含类定义。
    *   `path/to/file-tq-inl.inc`以包含在相应的内联标头中`path/to/file-inl.h`，包含类定义的C++访问器。
    *   `path/to/file-tq.cc`包含生成的堆验证程序、打印机等。

    扭矩编译器还生成各种其他已知的`.h`文件，意味着由 V8 内部版本使用。
2.  这`gn`然后生成编译生成的`-csa.cc`从步骤 1 到`mksnapshot`可执行。
3.  什么时候`mksnapshot`运行，V8的所有内置组件都会生成并打包到快照文件中，包括在Torque和任何其他使用Torque定义功能的内置组件中定义的那些。
4.  V8的其余部分已经建成。所有Torque编写的内置组件都可以通过链接到V8的快照文件进行访问。它们可以像任何其他内置的一样调用。此外，`d8`或`chrome`可执行文件还包括生成的编译单元，这些编译单元直接与类定义相关。

从图形上看，构建过程如下所示：

<figure>
  <img src="/_img/docs/torque/build-process.svg" width="800" height="480" alt="" loading="lazy">
</figure>

## 扭矩工具 { #tooling }

扭矩提供基本的工具和开发环境支持。

*   有一个[Visual Studio Code 插件](https://github.com/v8/vscode-torque)对于 Torque，它使用自定义语言服务器来提供 go-to-definition 等功能。
*   还有一个格式化工具，应该在更改后使用`.tq`文件：`tools/torque/format-torque.py -i <filename>`

## 涉及扭矩的故障排除构建 { #troubleshooting }

你为什么需要知道这一点？了解如何将 Torque 文件转换为机器代码非常重要，因为在将 Torque 转换为快照中嵌入的二进制位的不同阶段，可能会出现不同的问题（和错误）：

*   如果您在扭矩代码中存在语法或语义错误（即`.tq`文件），扭矩编译器失败。V8 构建在此阶段中止，您不会看到构建的后续部分可能发现的其他错误。
*   一旦您的扭矩代码在语法上正确并通过扭矩编译器（或多或少）严格的语义检查，构建`mksnapshot`仍然可能失败。这最常发生在`.tq`文件。标有`extern`扭矩代码中的关键字向扭矩编译器发出信号，表明所需功能的定义位于C++。目前，耦合间`extern`定义来自`.tq`文件和C++代码`extern`定义引用是松散的，并且在该耦合的扭矩编译时没有验证。什么时候`extern`定义与它们在`code-stub-assembler.h`头文件或其他 V8 头文件，C++构建`mksnapshot`失败。
*   甚至一次`mksnapshot`成功构建，在执行期间可能会失败。发生这种情况可能是涡轮风扇无法编译生成的 CSA 代码，例如，因为扭矩`static_assert`无法由涡轮风扇验证。此外，在快照创建期间运行的扭矩提供的内置可能存在错误。例如`Array.prototype.splice`，一个 Torque 编写的内置，作为 JavaScript 快照初始化过程的一部分调用，以设置默认的 JavaScript 环境。如果实现中存在错误，`mksnapshot`在执行过程中崩溃。什么时候`mksnapshot`崩溃，有时调用很有用`mksnapshot`通过`--gdb-jit-full`flag，它生成额外的调试信息，提供有用的上下文，例如扭矩生成的内置内存的名称`gdb`堆栈爬网。
*   当然，即使Torque编写的代码也能通过`mksnapshot`，它仍然可能是错误或崩溃。将测试用例添加到`torque-test.tq`和`torque-test.cc`是确保您的扭矩代码达到您实际期望的好方法。如果您的扭矩代码最终崩溃`d8`或`chrome`这`--gdb-jit-full`标志再次非常有用。

## `constexpr`：编译时与运行时 { #constexpr }

了解扭矩构建过程对于理解扭矩语言中的核心功能也很重要：`constexpr`.

Torque允许在运行时（即V8内置内容作为执行JavaScript的一部分执行时）评估Torque代码中的表达式。但是，它还允许在编译时执行表达式（即作为 Torque 构建过程的一部分，在 V8 库和`d8`可执行文件甚至已经创建）。

扭矩使用`constexpr`关键字，指示必须在生成时计算表达式。它的用法有点类似于[C++`constexpr`](https://en.cppreference.com/w/cpp/language/constexpr)：除了借用`constexpr`关键字及其一些语法来自C++，扭矩同样使用`constexpr`以指示编译时和运行时的评估之间的区别。

但是，扭矩有一些细微的差异`constexpr`语义学。在C++，`constexpr`表达式可以完全由C++编译器计算。扭矩`constexpr`Torque 编译器无法完全计算表达式，而是映射到C++类型、变量和表达式，这些类型、变量和表达式在运行时可以（并且必须）完全计算`mksnapshot`.从扭矩编写器的角度来看，`constexpr`表达式不生成在运行时执行的代码，因此从这个意义上说，它们是编译时的，即使它们在技术上由扭矩外部的C++代码进行评估，`mksnapshot`运行。因此，在扭矩中，`constexpr`本质上意味着”`mksnapshot`-time“，而不是”编译时”。

与泛型结合使用，`constexpr`是一个功能强大的Torque工具，可用于自动生成多个非常高效的专用内置组件，这些内置组件在V8开发人员可以提前预测的少量特定细节中彼此不同。

## 文件

扭矩代码打包在单独的源文件中。每个源文件都由一系列声明组成，这些声明本身可以选择包装在命名空间声明中以分隔声明的命名空间。以下语法描述可能已过时。事实的来源是[扭矩编译器中的语法定义](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/torque/torque-parser.cc?q=TorqueGrammar::TorqueGrammar)，它是使用无 contex 语法规则编写的。

扭矩文件是一系列声明。列出了可能的声明[在`torque-parser.cc`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/torque/torque-parser.cc?q=TorqueGrammar::declaration).

## 命名空间

扭矩命名空间允许声明位于独立的命名空间中。它们类似于C++命名空间。它们允许您创建在其他命名空间中不自动可见的声明。它们可以嵌套，嵌套命名空间中的声明可以无条件地访问包含它们的名称空间中的声明。未显式存在于命名空间声明中的声明将放在对所有命名空间可见的共享全局默认命名空间中。可以重新打开命名空间，从而允许在多个文件上定义命名空间。

例如：

```torque
macro IsJSObject(o: Object): bool { … }  // In default namespace

namespace array {
  macro IsJSArray(o: Object): bool { … }  // In array namespace
};

namespace string {
  // …
  macro TestVisibility() {
    IsJsObject(o); // OK, global namespace visible here
    IsJSArray(o);  // ERROR, not visible in this namespace
    array::IsJSArray(o);  // OK, explicit namespace qualification
  }
  // …
};

namespace array {
  // OK, namespace has been re-opened.
  macro EnsureWriteableFastElements(array: JSArray){ … }
};
```

## 声明

### 类型

扭矩类型强大。它的类型系统是它提供的许多安全性和正确性保证的基础。

对于许多基本类型，Torque实际上对它们本身并不是很了解。相反，许多类型只是松散地耦合`CodeStubAssembler`并通过显式类型映射C++类型，并依靠C++编译器来强制实施该映射的严格性。此类类型作为抽象类型实现。

#### 抽象类型

Torque的抽象类型直接映射到C++编译时和CodeStubAssembler运行时值。它们的声明指定名称和与C++类型的关系：

```grammar
AbstractTypeDeclaration :
  type IdentifierName ExtendsDeclaration opt GeneratesDeclaration opt ConstexprDeclaration opt

ExtendsDeclaration :
  extends IdentifierName ;

GeneratesDeclaration :
  generates StringLiteral ;

ConstexprDeclaration :
  constexpr StringLiteral ;
```

`IdentifierName`指定抽象类型的名称，以及`ExtendsDeclaration`（可选）指定声明的类型派生自的类型。`GeneratesDeclaration`（可选）指定与C++相对应的字符串文本`TNode`类型用于`CodeStubAssembler`代码以包含其类型的运行时值。`ConstexprDeclaration`是一个字符串文本，指定 C++与`constexpr`用于构建时的扭矩类型的版本（`mksnapshot`-时间） 评估。

下面是一个示例`base.tq`对于 Torque 的 31 位和 32 位有符号整数类型：

```torque
type int32 generates 'TNode<Int32T>' constexpr 'int32_t';
type int31 extends int32 generates 'TNode<Int32T>' constexpr 'int31_t';
```

#### 联合类型

联合类型表示值属于几种可能的类型之一。我们只允许标记值的联合类型，因为它们可以在运行时使用映射指针进行区分。例如，JavaScript 数字要么是 Smi 值，要么是已分配的`HeapNumber`对象。

```torque
type Number = Smi | HeapNumber;
```

联合类型满足以下相等性：

*   `A | B = B | A`
*   `A | (B | C) = (A | B) | C`
*   `A | B = A`如果`B`是 的子类型`A`

只允许从标记的类型形成联合类型，因为在运行时无法区分未标记的类型。

将联合类型映射到 CSA 时，将选择联合类型的所有类型中最具体的通用超类型，但`Number`和`Numeric`，它们映射到相应的 CSA 联合类型。

#### 类类型

类类型使得从扭矩代码定义，分配和操作V8 GC堆上的结构化对象成为可能。每个扭矩类类型都必须对应于C++代码中的堆对象的子类。为了最大限度地减少在 V8 的 C++ 和 Torque 实现之间维护样板对象访问代码的费用，Torque 类定义用于在可能（和适当）的情况下生成所需的C++对象访问代码，以减少手动保持C++和 Torque 同步的麻烦。

```grammar
ClassDeclaration :
  ClassAnnotation* extern opt transient opt class IdentifierName ExtendsDeclaration opt GeneratesDeclaration opt {
    ClassMethodDeclaration*
    ClassFieldDeclaration*
  }

ClassAnnotation :
  @doNotGenerateCppClass
  @generateBodyDescriptor
  @generatePrint
  @abstract
  @export
  @noVerifier
  @hasSameInstanceTypeAsParent
  @highestInstanceTypeWithinParentClassRange
  @lowestInstanceTypeWithinParentClassRange
  @reserveBitsInInstanceType ( NumericLiteral )
  @apiExposedInstanceTypeValue ( NumericLiteral )

ClassMethodDeclaration :
  transitioning opt IdentifierName ImplicitParameters opt ExplicitParameters ReturnType opt LabelsDeclaration opt StatementBlock

ClassFieldDeclaration :
  ClassFieldAnnotation* weak opt const opt FieldDeclaration;

ClassFieldAnnotation :
  @noVerifier
  @if ( Identifier )
  @ifnot ( Identifier )

FieldDeclaration :
  Identifier ArraySpecifier opt : Type ;

ArraySpecifier :
  [ Expression ]
```

示例类：

```torque
extern class JSProxy extends JSReceiver {
  target: JSReceiver|Null;
  handler: JSReceiver|Null;
}
```

`extern`表示此类是在 C++ 中定义的，而不是仅在 Torque 中定义的。

类中的字段声明隐式生成可从 CodeStubAssembler 中使用的字段 getter 和 setter，例如：

```cpp
// In TorqueGeneratedExportedMacrosAssembler:
TNode<HeapObject> LoadJSProxyTarget(TNode<JSProxy> p_o);
void StoreJSProxyTarget(TNode<JSProxy> p_o, TNode<HeapObject> p_v);
```

如上所述，Torque 类中定义的字段生成C++代码，无需重复的样板访问器和堆访问者代码。JSProxy 的手写定义必须继承自生成的类模板，如下所示：

```cpp
// In js-proxy.h:
class JSProxy : public TorqueGeneratedJSProxy<JSProxy, JSReceiver> {

  // Whatever the class needs beyond Torque-generated stuff goes here...

  // At the end, because it messes with public/private:
  TQ_OBJECT_CONSTRUCTORS(JSProxy)
}

// In js-proxy-inl.h:
TQ_OBJECT_CONSTRUCTORS_IMPL(JSProxy)
```

生成的类提供强制转换函数、字段访问器函数和字段偏移常量（例如`kTargetOffset`和`kHandlerOffset`在本例中），表示从类的开头开始的每个字段的字节偏移量。

##### 类类型注记

某些类不能使用上述示例中所示的继承模式。在这些情况下，类可以指定`@doNotGenerateCppClass`，直接从其超类类型继承，并为其场偏移常量包含扭矩生成的宏。此类类必须实现自己的访问器和强制转换函数。使用该宏如下所示：

```cpp
class JSProxy : public JSReceiver {
 public:
  DEFINE_FIELD_OFFSET_CONSTANTS(
      JSReceiver::kHeaderSize, TORQUE_GENERATED_JS_PROXY_FIELDS)
  // Rest of class omitted...
}
```

`@generateBodyDescriptor`导致扭矩发出类`BodyDescriptor`在生成的类中，该类表示垃圾回收器应如何访问对象。否则，C++代码必须定义自己的对象访问，或使用现有模式之一（例如，从`Struct`并将类包含在`STRUCT_LIST`意味着该类应仅包含标记的值）。

如果`@generatePrint`添加注释，然后生成器将实现一个C++函数，该函数打印由扭矩布局定义的字段值。使用 JSProxy 示例，签名将为`void TorqueGeneratedJSProxy<JSProxy, JSReceiver>::JSProxyPrint(std::ostream& os)`，可以由 继承`JSProxy`.

扭矩编译器还为所有`extern`类，除非该类选择退出`@noVerifier`注解。例如，上面的 JSProxy 类定义将生成一个C++方法`void TorqueGeneratedClassVerifiers::JSProxyVerify(JSProxy o, Isolate* isolate)`根据扭矩类型定义验证其字段是否有效。它还将在生成的类上生成相应的函数，`TorqueGeneratedJSProxy<JSProxy, JSReceiver>::JSProxyVerify`，它调用静态函数`TorqueGeneratedClassVerifiers`.如果要为类添加额外的验证（例如，数字上可接受的值的范围，或要求该字段`foo`如果字段为真`bar`为非空值等），然后添加`DECL_VERIFIER(JSProxy)`到C++类（隐藏继承的`JSProxyVerify`）， 并将其实现于`src/objects-debug.cc`.任何此类自定义验证程序的第一步都应该是调用生成的验证程序，例如`TorqueGeneratedClassVerifiers::JSProxyVerify(*this, isolate);`.（要在每个 GC 之前和之后运行这些验证程序，请使用`v8_enable_verify_heap = true`并运行`--verify-heap`.)

`@abstract`表示类本身未实例化，并且没有自己的实例类型：逻辑上属于该类的实例类型是派生类的实例类型。

这`@export`注释使 Torque 编译器生成具体的C++类（如`JSProxy`在上面的示例中）。显然，这只有在您不想添加扭矩生成代码提供的功能之外的任何C++功能时才有用。不能与`extern`.对于仅在 Torque 中定义和使用的类，最合适的做法是两者都不使用`extern`也不`@export`.

`@hasSameInstanceTypeAsParent`表示与其父类具有相同实例类型，但重命名某些字段或可能具有不同映射的类。在这种情况下，父类不是抽象的。

批注`@highestInstanceTypeWithinParentClassRange`,`@lowestInstanceTypeWithinParentClassRange`,`@reserveBitsInInstanceType`和`@apiExposedInstanceTypeValue`所有这些都会影响实例类型的生成。一般来说，你可以忽略这些，没关系。扭矩负责在枚举中分配唯一值`v8::internal::InstanceType`，以便 V8 可以在运行时确定 JS 堆中任何对象的类型。在绝大多数情况下，Torque的实例类型分配应该是足够的，但是在少数情况下，我们希望特定类的实例类型在构建之间保持稳定，或者位于分配给其超类的实例类型范围的开头或结尾，或者成为可以在Torque之外定义的保留值范围。

##### 类字段

除了纯值（如上例所示），类字段还可以包含索引数据。下面是一个示例：

```torque
extern class CoverageInfo extends HeapObject {
  const slot_count: int32;
  slots[slot_count]: CoverageInfoSlot;
}
```

这意味着`CoverageInfo`根据 中的数据具有不同的大小`slot_count`.

与C++不同，Torque不会在字段之间隐式添加填充;相反，如果字段未正确对齐，它将失败并发出错误。扭矩还要求强场、弱场和标量场与场顺序中同一类别的其他场一起。

`const`意味着字段不能在运行时更改（或者至少不容易更改）;如果您尝试设置扭矩，则编译将失败）。对于长度字段来说，这是一个好主意，应该非常小心地重置长度字段，因为它们需要释放任何释放的空间，并且可能导致带有标记线程的数据争用。
实际上，扭矩要求用于索引数据的长度字段为`const`.

`weak`在字段声明的开头意味着该字段是自定义的弱引用，而不是`MaybeObject`弱场的标记机制。
另外`weak`影响常量的生成，例如`kEndOfStrongFieldsOffset`和`kStartOfWeakFieldsOffset`，这是某些自定义中使用的旧功能`BodyDescriptor`，并且当前仍需要将字段分组为`weak`一起。我们希望在扭矩完全能够生成所有关键字后删除此关键字`BodyDescriptor`s.

如果存储在字段中的对象可能是`MaybeObject`-样式弱引用（设置第二位），然后`Weak<T>`应用于类型和`weak`关键字应**不**被使用。此规则仍有一些例外，例如此字段`Map`，其中可以包含一些强类型和一些弱类型，并且还标记为`weak`包含在弱部分中：

```torque
  weak transitions_or_prototype_info: Map|Weak<Map>|TransitionArray|
      PrototypeInfo|Smi;
```

`@if`和`@ifnot`标记应包含在某些生成配置中但不应包含在其他生成配置中的字段。它们接受 列表中的值`BuildFlags`在`src/torque/torque-parser.cc`.

##### 完全在扭矩之外定义的类别

某些类未在 Torque 中定义，但 Torque 必须了解每个类，因为它负责分配实例类型。对于这种情况，类可以在没有实体的情况下声明，并且Torque除了实例类型之外不会为它们生成任何内容。例：

```torque
extern class OrderedHashMap extends HashTable;
```

#### 形状

定义`shape`看起来就像定义一个`class`除了它使用关键字`shape`而不是`class`.一个`shape`是 的子类型`JSObject`表示对象内属性的时间点排列（在规范中，这些是“数据属性”而不是“内部槽”）。一个`shape`没有自己的实例类型。具有特定形状的对象可能随时更改并丢失该形状，因为该对象可能会进入字典模式并将其所有属性移出到单独的后备存储中。

#### 结构件

`struct`s 是可以轻松一起传递的数据集合。（与名为的类完全无关`Struct`.)与类一样，它们可以包含对数据进行操作的宏。与类不同，它们还支持泛型。语法类似于类：

```torque
@export
struct PromiseResolvingFunctions {
  resolve: JSFunction;
  reject: JSFunction;
}

struct ConstantIterator<T: type> {
  macro Empty(): bool {
    return false;
  }
  macro Next(): T labels _NoMore {
    return this.value;
  }

  value: T;
}
```

##### 结构批注

任何标记为`@export`将在生成的文件中包含可预测的名称`gen/torque-generated/csa-types.h`.名称前面附加了`TorqueStruct`所以`PromiseResolvingFunctions`成为`TorqueStructPromiseResolvingFunctions`.

结构字段可以标记为`const`，这意味着不应写入它们。整个结构仍然可以被覆盖。

##### 结构作为类字段

结构可以用作类字段的类型。在这种情况下，它表示类中打包的有序数据（否则，结构没有对齐要求）。这对于类中的索引字段特别有用。例如，`DescriptorArray`包含一个三值结构数组：

```torque
struct DescriptorEntry {
  key: Name|Undefined;
  details: Smi|Undefined;
  value: JSAny|Weak<Map>|AccessorInfo|AccessorPair|ClassPositions;
}

extern class DescriptorArray extends HeapObject {
  const number_of_all_descriptors: uint16;
  number_of_descriptors: uint16;
  raw_number_of_marked_descriptors: uint16;
  filler16_bits: uint16;
  enum_cache: EnumCache;
  descriptors[number_of_all_descriptors]: DescriptorEntry;
}
```

##### 引用和切片

`Reference<T>`和`Slice<T>`是表示指向堆对象中保存的数据的指针的特殊结构。它们都包含一个对象和一个偏移量;`Slice<T>`还包含长度。您可以使用特殊语法，而不是直接构造这些结构：`&o.x`将创建一个`Reference`到现场`x`在对象内`o`，或`Slice`在以下情况下`x`是索引字段。对于引用和切片，都有常量和可变版本。对于引用，这些类型编写为`&T`和`const &T`分别用于可变引用和常量引用。可变性是指它们指向的数据，可能不会全局保存，也就是说，您可以创建对可变数据的 const 引用。对于切片，类型没有特殊语法，而是编写两个版本`ConstSlice<T>`和`MutableSlice<T>`.引用可以通过以下方式取消引用`*`或`->`，与C++一致。

对未标记数据的引用和切片也可以指向堆外数据。

#### 比特菲尔德结构

一个`bitfield struct`表示打包到单个数值中的数值数据的集合。它的语法看起来类似于正常语法`struct`，并添加每个字段的位数。

```torque
bitfield struct DebuggerHints extends uint31 {
  side_effect_state: int32: 2 bit;
  debug_is_blackboxed: bool: 1 bit;
  computed_debug_is_blackboxed: bool: 1 bit;
  debugging_id: int32: 20 bit;
}
```

如果位字段结构（或任何其他数值数据）存储在 Smi 中，则可以使用以下类型来表示它`SmiTagged<T>`.

#### 函数指针类型

函数指针只能指向 Torque 中定义的内置函数，因为这保证了默认的 ABI。它们对于减小二进制代码大小特别有用。

虽然函数指针类型是匿名的（如在 C 中），但它们可以绑定到类型别名（如`typedef`在C中）。

```torque
type CompareBuiltinFn = builtin(implicit context: Context)(Object, Object, Object) => Number;
```

#### 特殊类型

关键字指示有两种特殊类型`void`和`never`.`void`用作不返回值的可调用的返回类型，并且`never`用作从未实际返回的可调用的返回类型（即仅通过异常路径退出）。

#### 瞬态类型

在 V8 中，堆对象可以在运行时更改布局。为了表达类型系统中可能发生变化或其他临时假设的对象布局，Torque支持“瞬态类型”的概念。声明抽象类型时，添加关键字`transient`将其标记为瞬态类型。

```torque
// A HeapObject with a JSArray map, and either fast packed elements, or fast
// holey elements when the global NoElementsProtector is not invalidated.
transient type FastJSArray extends JSArray
    generates 'TNode<JSArray>';
```

例如，在以下情况下`FastJSArray`，如果数组更改为字典元素或如果全局`NoElementsProtector`无效。要在 Torque 中表达这一点，请将所有可能这样做的可调用对象注释为`transitioning`.例如，调用 JavaScript 函数可以执行任意 JavaScript，因此它是`transitioning`.

```torque
extern transitioning macro Call(implicit context: Context)
                               (Callable, Object): Object;
```

在类型系统中对此进行监视的方式是，在转换操作中访问瞬态类型的值是非法的。

```torque
const fastArray : FastJSArray = Cast<FastJSArray>(array) otherwise Bailout;
Call(f, Undefined);
return fastArray; // Type error: fastArray is invalid here.
```

#### 枚举

枚举提供了一种定义一组常量并将它们分组到类似于
C++中的枚举类。声明由`enum`关键字并坚持以下
语法结构：

```grammar
EnumDeclaration :
  extern enum IdentifierName ExtendsDeclaration opt ConstexprDeclaration opt { IdentifierName list+ (, ...) opt }
```

一个基本示例如下所示：

```torque
extern enum LanguageMode extends Smi {
  kStrict,
  kSloppy
}
```

此声明定义一个新类型`LanguageMode`，其中`extends`子句指定基础
type，即用于表示枚举值的运行时类型。在此示例中，这是`TNode<Smi>`,
因为这是什么类型`Smi` `generates`.一个`constexpr LanguageMode`转换为`LanguageMode`
在生成的 CSA 文件中，因为没有`constexpr`在枚举中指定子句以替换缺省名称。
如果`extends`省略子句，扭矩将仅生成`constexpr`类型的版本。这`extern`关键字告诉 Torque，此枚举有一个C++定义。目前，仅`extern`支持枚举。

扭矩为每个枚举的条目生成不同的类型和常数。这些是定义的
在与枚举名称匹配的命名空间中。必要的专业化`FromConstexpr<>`是
生成以从条目的`constexpr`类型到枚举类型。为C++文件中的条目生成的值为`<enum-constexpr>::<entry-name>`哪里`<enum-constexpr>`是`constexpr`为枚举生成的名称。在上面的例子中，这些是`LanguageMode::kStrict`和`LanguageMode::kSloppy`.

扭矩的枚举与`typeswitch`构造，因为
值是使用不同的类型定义的：

```torque
typeswitch(language_mode) {
  case (LanguageMode::kStrict): {
    // ...
  }
  case (LanguageMode::kSloppy): {
    // ...
  }
}
```

如果枚举的C++定义包含的值多于`.tq`文件，扭矩需要知道。这是通过附加一个`...`在最后一个条目之后。考虑`ExtractFixedArrayFlag`例如，只有部分选项在内部可用/使用
力矩：

```torque
enum ExtractFixedArrayFlag constexpr 'CodeStubAssembler::ExtractFixedArrayFlag' {
  kFixedDoubleArrays,
  kAllFixedArrays,
  kFixedArrays,
  ...
}
```

### 可调用

可调用在概念上类似于JavaScript或C++中的函数，但它们具有一些额外的语义，允许它们以有用的方式与CSA代码和V8运行时进行交互。Torque提供了几种不同类型的可调用对象：`macro`s,`builtin`s,`runtime`s 和`intrinsic`s.

```grammar
CallableDeclaration :
  MacroDeclaration
  BuiltinDeclaration
  RuntimeDeclaration
  IntrinsicDeclaration
```

#### `macro`可调用

宏是一个可调用的，对应于生成的 CSA 生成C++块。`macro`s 可以在 Torque 中完全定义，在这种情况下，CSA 代码由 Torque 生成，或者标记为`extern`，在这种情况下，必须在 CodeStubAssembler 类中以手写的 CSA 代码的形式提供实现。从概念上讲，考虑这一点是有用的`macro`在调用站点上内联的可内联 CSA 代码块。

`macro`扭矩中的声明采用以下形式：

```grammar
MacroDeclaration :
   transitioning opt macro IdentifierName ImplicitParameters opt ExplicitParameters ReturnType opt LabelsDeclaration opt StatementBlock
  extern transitioning opt macro IdentifierName ImplicitParameters opt ExplicitTypes ReturnType opt LabelsDeclaration opt ;
```

每个非`extern`力矩`macro`使用`StatementBlock`正文`macro`在其生成的命名空间中创建 CSA 生成函数`Assembler`类。此代码看起来就像您可能在其中找到的其他代码一样`code-stub-assembler.cc`，尽管可读性有点差，因为它是机器生成的。`macro`标记`extern`没有用Torque编写的正文，只是提供手写C++CSA代码的接口，以便可以从Torque使用。

`macro`定义指定隐式和显式参数、可选返回类型和可选标签。下面将更详细地讨论参数和返回类型，但现在只需知道它们的工作方式有点像 TypeScript 参数，如 TypeScript 文档的“函数类型”部分所述。[这里](https://www.typescriptlang.org/docs/handbook/functions.html).

标签是一种特殊退出的机制`macro`.它们以 1：1 的比例映射到 CSA 标签，并添加为`CodeStubAssemblerLabels*`-类型化参数 C++到为`macro`.它们的确切语义将在下面讨论，但为了`macro`声明，逗号分隔的列表`macro`的标签可选择与`labels`关键字和位置在`macro`的参数列表和返回类型。

下面是一个示例`base.tq`外部和扭矩定义`macro`s:

```torque
extern macro BranchIfFastJSArrayForCopy(Object, Context): never
    labels Taken, NotTaken;
macro BranchIfNotFastJSArrayForCopy(implicit context: Context)(o: Object):
    never
    labels Taken, NotTaken {
  BranchIfFastJSArrayForCopy(o, context) otherwise NotTaken, Taken;
}
```

#### `builtin`可调用

`builtin`s 类似于`macro`s，因为它们可以在扭矩中完全定义或标记为`extern`.在基于扭矩的内置情况下，内置的主体用于生成V8内置，可以像任何其他V8内置一样调用，包括自动添加相关信息`builtin-definitions.h`.喜欢`macro`s， 扭矩`builtin`标记`extern`没有基于扭矩的车身，只需提供与现有V8的接口`builtin`s，以便可以从扭矩代码中使用它们。

`builtin`扭矩中的声明具有以下形式：

```grammar
MacroDeclaration :
  transitioning opt javascript opt builtin IdentifierName ImplicitParameters opt ExplicitParametersOrVarArgs ReturnType opt StatementBlock
  extern transitioning opt javascript opt builtin IdentifierName ImplicitParameters opt ExplicitTypesOrVarArgs ReturnType opt ;
```

Torque内置的代码只有一个副本，那就是在生成的内置代码对象中。与`macro`s，当`builtin`s是从扭矩代码调用的，CSA代码不是在调用站点内联的，而是生成对内置的调用。

`builtin`s 不能有标签。

如果您正在编写一个`builtin`，你可以制作一个[尾声呼叫](https://en.wikipedia.org/wiki/Tail_call)对于内置或运行时函数iff（当且仅当），它是内置中的最终调用。在这种情况下，编译器可能能够避免创建新的堆栈帧。只需添加`tail`在呼叫之前，如`tail MyBuiltin(foo, bar);`.

#### `runtime`可调用

`runtime`s 类似于`builtin`s，因为它们可以将外部功能的接口公开给Torque。但是，不是在 CSA 中实现，而是由`runtime`必须始终在 V8 中作为标准运行时回调实现。

`runtime`扭矩中的声明具有以下形式：

```grammar
MacroDeclaration :
  extern transitioning opt runtime IdentifierName ImplicitParameters opt ExplicitTypesOrVarArgs ReturnType opt ;
```

这`extern runtime`使用名称指定<i>标识符名称</i>对应于<code>运行时：：k<i>标识符名称</i></code>.

喜欢`builtin`s,`runtime`s 不能有标签。

您也可以致电`runtime`在适当的时候充当尾随器。只需包含`tail`调用前的关键字。

运行时函数声明通常放置在名为`runtime`.这消除了它们与同名内置组件的歧义，并且更容易在调用站点上看到我们称之为运行时功能的内容。我们应该考虑将其作为强制性的。

#### `intrinsic`可调用

`intrinsic`s 是内置的 Torque 可调用，提供对内部功能的访问，而这些功能无法在 Torque 中实现。它们在 Torque 中声明，但未定义，因为实现由 Torque 编译器提供。`intrinsic`声明使用以下语法：

```grammar
IntrinsicDeclaration :
  intrinsic % IdentifierName ImplicitParameters opt ExplicitParameters ReturnType opt ;
```

在大多数情况下，“用户”扭矩代码应该很少使用`intrinsic`直接。
以下是一些受支持的内部函数：

```torque
// %RawObjectCast downcasts from Object to a subtype of Object without
// rigorous testing if the object is actually the destination type.
// RawObjectCasts should *never* (well, almost never) be used anywhere in
// Torque code except for in Torque-based UnsafeCast operators preceeded by an
// appropriate type assert()
intrinsic %RawObjectCast<A: type>(o: Object): A;

// %RawPointerCast downcasts from RawPtr to a subtype of RawPtr without
// rigorous testing if the object is actually the destination type.
intrinsic %RawPointerCast<A: type>(p: RawPtr): A;

// %RawConstexprCast converts one compile-time constant value to another.
// Both the source and destination types should be 'constexpr'.
// %RawConstexprCast translate to static_casts in the generated C++ code.
intrinsic %RawConstexprCast<To: type, From: type>(f: From): To;

// %FromConstexpr converts a constexpr value into into a non-constexpr
// value. Currently, only conversion to the following non-constexpr types
// are supported: Smi, Number, String, uintptr, intptr, and int32
intrinsic %FromConstexpr<To: type, From: type>(b: From): To;

// %Allocate allocates an unitialized object of size 'size' from V8's
// GC heap and "reinterpret casts" the resulting object pointer to the
// specified Torque class, allowing constructors to subsequently use
// standard field access operators to initialize the object.
// This intrinsic should never be called from Torque code. It's used
// internally when desugaring the 'new' operator.
intrinsic %Allocate<Class: type>(size: intptr): Class;
```

喜欢`builtin`s 和`runtime`s,`intrinsic`s 不能有标签。

### 显式参数

扭矩定义的可调用对象的声明，例如扭矩`macro`s 和`builtin`s，具有显式参数列表。它们是标识符和类型对的列表，使用的语法让人联想到类型化的 TypeScript 函数参数列表，但 Torque 不支持可选参数或默认参数。此外，扭矩执行`builtin`如果内置使用 V8 的内部 JavaScript 调用约定（例如，标记为`javascript`关键字）。

```grammar
ExplicitParameters :
  ( ( IdentifierName : TypeIdentifierName ) list* )
  ( ( IdentifierName : TypeIdentifierName ) list+ (, ... IdentifierName ) opt )
```

例如：

```torque
javascript builtin ArraySlice(
    (implicit context: Context)(receiver: Object, ...arguments): Object {
  // …
}
```

### 隐式参数

扭矩可调用对象可以使用类似于[Scala 的隐式参数](https://docs.scala-lang.org/tour/implicit-parameters.html):

```grammar
ImplicitParameters :
  ( implicit ( IdentifierName : TypeIdentifierName ) list* )
```

具体而言：A`macro`除了显式参数之外，还可以声明隐式参数：

```torque
macro Foo(implicit context: Context)(x: Smi, y: Smi)
```

映射到 CSA 时，隐式参数和显式参数被视为相同，并形成联合参数列表。

隐式参数在调用站点中未提及，而是隐式传递：`Foo(4, 5)`.为了做到这一点，`Foo(4, 5)`必须在提供名为`context`.例：

```torque
macro Bar(implicit context: Context)() {
  Foo(4, 5);
}
```

与 Scala 相反，如果隐式参数的名称不相同，我们禁止这样做。

由于重载分辨率会导致混淆行为，因此我们确保隐式参数完全不会影响重载分辨率。也就是说：在比较重载集的候选项时，我们不考虑调用站点上可用的隐式绑定。只有在找到一个最佳重载之后，我们才会检查隐式参数的隐式绑定是否可用。

将隐式参数保留在显式参数中与 Scala 不同，但更好地映射到 CSA 中的现有约定，以`context`参数优先。

#### `js-implicit`

对于在 Torque 中定义了 JavaScript 链接的内置组件，您应该使用关键字`js-implicit`而不是`implicit`.参数仅限于调用约定的以下四个组件：

*   上下文：`NativeContext`
*   接收器：`JSAny`(`this`在 JavaScript 中）
*   目标：`JSFunction`(`arguments.callee`在 JavaScript 中）
*   新目标：`JSAny`(`new.target`在 JavaScript 中）

它们不必全部声明，只需声明要使用的那些。例如，下面是我们的代码`Array.prototype.shift`:

```torque
  // https://tc39.es/ecma262/#sec-array.prototype.shift
  transitioning javascript builtin ArrayPrototypeShift(
      js-implicit context: NativeContext, receiver: JSAny)(...arguments): JSAny {
  ...
```

请注意，`context`参数是一个`NativeContext`.这是因为 V8 中的内置内容始终在其闭包中嵌入本机上下文。在 js 隐式约定中对此进行编码允许程序员消除从函数上下文中加载本机上下文的操作。

### 过载分辨率

力矩`macro`s 和运算符（它们只是`macro`s） 允许参数类型重载。重载规则的灵感来自C++：如果重载严格优于所有替代方案，则选择重载。这意味着它必须在至少一个参数中严格更好，而在所有其他参数中必须更好或同样好。

当比较两个过载的一对对应参数时...

*   ...如果出现以下情况，它们被认为同样好：
    *   他们是平等的;
    *   两者都需要一些隐式转换。
*   ...如果出现以下情况，则认为一种更好：
    *   它是另一个的严格子类型;
    *   它不需要隐式转换，而另一个则需要隐式转换。

如果没有重载严格优于所有备选项，则会导致编译错误。

### 延迟块

可以选择将语句块标记为`deferred`，这是向编译器发出的信号，表明它输入的频率较低。编译器可以选择将这些块放在函数的末尾，从而提高代码的非延迟区域的缓存局部性。例如，在此代码中，从`Array.prototype.forEach`实施后，我们期望保持在“快速”的道路上，只是很少采取救助案例：

```torque
  let k: Number = 0;
  try {
    return FastArrayForEach(o, len, callbackfn, thisArg)
        otherwise Bailout;
  }
  label Bailout(kValue: Smi) deferred {
    k = kValue;
  }
```

下面是另一个示例，其中字典元素事例标记为延迟，以改进更可能情况的代码生成（从`Array.prototype.join`实现）：

```torque
  if (IsElementsKindLessThanOrEqual(kind, HOLEY_ELEMENTS)) {
    loadFn = LoadJoinElement<FastSmiOrObjectElements>;
  } else if (IsElementsKindLessThanOrEqual(kind, HOLEY_DOUBLE_ELEMENTS)) {
    loadFn = LoadJoinElement<FastDoubleElements>;
  } else if (kind == DICTIONARY_ELEMENTS)
    deferred {
      const dict: NumberDictionary =
          UnsafeCast<NumberDictionary>(array.elements);
      const nofElements: Smi = GetNumberDictionaryNumberOfElements(dict);
      // <etc>...
```

## 将 CSA 代码移植到扭矩

[移植的修补程序`Array.of`](https://chromium-review.googlesource.com/c/v8/v8/+/1296464)作为将 CSA 代码移植到 Torque 的最小示例。
