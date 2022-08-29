***

title： “使用新的类功能更快地初始化实例”
作者： '[张嘉仪](https://twitter.com/JoyeeCheung)、实例初始值设定项
化身：

*   “乔伊祥”
    日期： 2022-04-20
    标签：
*   内部
    描述： “自 V8 v9.7 以来，具有新类功能的实例初始化速度变得更快。
    推文：“1517041137378373632”

***

自 v7.2 以来，类字段已在 V8 中提供，私有类方法已从 v8.4 开始提供。在提案于2021年达到第4阶段后，已经开始努力改善V8中对新类功能的支持 - 在此之前，有两个主要问题影响了它们的采用：

1.  类字段和私有方法的初始化比普通属性的赋值慢得多。
2.  类字段初始值设定项在[启动快照](https://v8.dev/blog/custom-startup-snapshots)由 Node.js 和 Deno 等嵌入器使用，以加快自身或用户应用程序的引导速度。

第一个问题已在 V8 v9.7 中修复，第二个问题的修复已在 V8 v10.0 中发布。这篇文章介绍了如何修复第一个问题，有关快照问题修复的另一篇文章，请查看[这篇文章](https://joyeecheung.github.io/blog/2022/04/14/fixing-snapshot-support-of-class-fields-in-v8/).

## 优化类字段

为了消除普通属性的分配和类字段的初始化之间的性能差距，我们更新了现有的[内联高速缓存 （IC） 系统](https://mathiasbynens.be/notes/shapes-ics)与后者一起工作。在 v9.7 之前，V8 始终使用成本高昂的运行时调用来初始化类字段。在 v9.7 中，当 V8 认为初始化模式足够可预测时，它使用新的 IC 来加快操作速度，就像它对普通属性的分配所做的那样。

![Performance of initializations, optimized](../_img/faster-class-features/class-fields-performance-optimized.svg)

![Performance of initializations, interpreted](../_img/faster-class-features/class-fields-performance-interpreted.svg)

### 类字段的原始实现

为了实现私有字段，V8 利用了内部私有符号 — 它们是类似于标准的内部 V8 数据结构`Symbol`s，除非用作属性键时不可枚举。以这个类为例：

```js
class A {
  #a = 0;
  b = this.#a;
}
```

V8 将收集类字段初始值设定项 （`#a = 0`和`b = this.#a`）， 并生成一个综合实例成员函数，其中初始值设定项作为函数体。为此合成函数生成的字节码过去如下所示：

```cpp
// Load the private name symbol for `#a` into r1
LdaImmutableCurrentContextSlot [2]
Star r1

// Load 0 into r2
LdaZero
Star r2

// Move the target into r0
Mov <this>, r0

// Use the %AddPrivateField() runtime function to store 0 as the value of
// the property keyed by the private name symbol `#a` in the instance,
// that is, `#a = 0`.
CallRuntime [AddPrivateField], r0-r2

// Load the property name `b` into r1
LdaConstant [0]
Star r1

// Load the private name symbol for `#a`
LdaImmutableCurrentContextSlot [2]

// Load the value of the property keyed by `#a` from the instance into r2
LdaKeyedProperty <this>, [0]
Star r2

// Move the target into r0
Mov <this>, r0

// Use the %CreateDataProperty() runtime function to store the property keyed
// by `#a` as the value of the property keyed by `b`, that is, `b = this.#a`
CallRuntime [CreateDataProperty], r0-r2
```

将上一个代码段中的类与类似如下的类进行比较：

```js
class A {
  constructor() {
    this._a = 0;
    this.b = this._a;
  }
}
```

从技术上讲，这两个类并不等同，甚至忽略了两者之间可见性的差异。`this.#a`和`this._a`.该规范要求“定义”语义而不是“设置”语义。也就是说，类字段的初始化不会触发 setter 或`set`代理陷阱。因此，第一类的近似值应使用`Object.defineProperty()`而不是简单的赋值来初始化属性。此外，如果实例中已存在私有字段，则应抛出（如果正在初始化的目标在基构造函数中被覆盖为另一个实例）：

```js
class A {
  constructor() {
    // What the %AddPrivateField() call roughly translates to:
    const _a = %PrivateSymbol('#a')
    if (_a in this) {
      throw TypeError('Cannot initialize #a twice on the same object');
    }
    Object.defineProperty(this, _a, {
      writable: true,
      configurable: false,
      enumerable: false,
      value: 0
    });
    // What the %CreateDataProperty() call roughly translates to:
    Object.defineProperty(this, 'b', {
      writable: true,
      configurable: true,
      enumerable: true,
      value: this[_a]
    });
  }
}
```

为了在提案最终确定之前实现指定的语义，V8 使用了对运行时函数的调用，因为它们更灵活。如上面的字节码所示，公共字段的初始化是通过以下方式实现的：`%CreateDataProperty()`运行时调用，而私有字段的初始化是通过以下方式实现的`%AddPrivateField()`.由于调用运行时会产生大量开销，因此与普通对象属性的分配相比，类字段的初始化要慢得多。

然而，在大多数用例中，语义差异是微不足道的。在这些情况下，最好能获得优化属性分配的性能，因此在提案最终确定后创建了更优化的实现。

### 优化私有类字段和计算的公共类字段

为了加快私有类字段和计算公共类字段的初始化，该实现引入了一个新机制来插入[内联高速缓存 （IC） 系统](https://mathiasbynens.be/notes/shapes-ics)处理这些操作时。这种新机器有三个合作部分：

*   在字节码生成器中，新的字节码`DefineKeyedOwnProperty`.这在为`ClassLiteral::Property`表示类字段初始值设定项的 AST 节点。
*   在 TurboFan JIT 中，相应的 IR 操作码`JSDefineKeyedOwnProperty`，可以从新的字节码编译。
*   在IC系统中，一个新的`DefineKeyedOwnIC`用于新字节码的解释器处理程序以及从新 IR 操作码编译的代码。为了简化实现，新IC重用了其中的一些代码。`KeyedStoreIC`这是为普通物业商店准备的。

现在，当 V8 遇到此类时：

```js
class A {
  #a = 0;
}
```

它为初始值设定项生成以下字节码`#a = 0`:

```cpp
// Load the private name symbol for `#a` into r1
LdaImmutableCurrentContextSlot [2]
Star0

// Use the DefineKeyedOwnProperty bytecode to store 0 as the value of
// the property keyed by the private name symbol `#a` in the instance,
// that is, `#a = 0`.
LdaZero
DefineKeyedOwnProperty <this>, r0, [0]
```

当初始值设定项执行足够多次时，V8 会分配一个[反馈向量槽](https://ponyfoo.com/articles/an-introduction-to-speculative-optimization-in-v8)对于要初始化的每个字段。插槽包含要添加的字段的密钥（对于私有字段，则为私有名称符号）和一对[隐藏类](https://v8.dev/docs/hidden-classes)由于字段初始化，实例已在其中转换。在后续初始化中，IC 使用反馈来查看字段是否在具有相同隐藏类的实例上以相同的顺序初始化。如果初始化与 V8 之前看到的模式匹配（通常是这种情况），V8 将采用快速路径并使用预生成的代码执行初始化，而不是调用运行时，从而加快操作速度。如果初始化与 V8 之前看到的模式不匹配，它将回退到运行时调用来处理慢速情况。

### 优化命名公共类字段

为了加快命名公共类字段的初始化，我们重用了现有的`DefineNamedOwnProperty`调用`DefineNamedOwnIC`无论是在解释器中，还是通过从`JSDefineNamedOwnProperty`红外操作码。

现在，当 V8 遇到此类时：

```js
class A {
  #a = 0;
  b = this.#a;
}
```

它为`b = this.#a`初始 化：

```cpp
// Load the private name symbol for `#a`
LdaImmutableCurrentContextSlot [2]

// Load the value of the property keyed by `#a` from the instance into r2
// Note: LdaKeyedProperty is renamed to GetKeyedProperty in the refactoring
GetKeyedProperty <this>, [2]

// Use the DefineKeyedOwnProperty bytecode to store the property keyed
// by `#a` as the value of the property keyed by `b`, that is, `b = this.#a;`
DefineNamedOwnProperty <this>, [0], [4]
```

原创`DefineNamedOwnIC`机器不能简单地插入到命名公共类字段的处理中，因为它最初仅用于对象文字初始化。以前，它期望正在初始化的目标是自创建以来用户尚未触及的对象，这对于对象文本始终为真，但是当类扩展其构造函数覆盖目标的基类时，可以在用户定义的对象上初始化类字段：

```js
class A {
  constructor() {
    return new Proxy(
      { a: 1 },
      {
        defineProperty(object, key, desc) {
          console.log('object:', object);
          console.log('key:', key);
          console.log('desc:', desc);
          return true;
        }
      });
  }
}

class B extends A {
  a = 2;
  #b = 3;  // Not observable.
}

// object: { a: 1 },
// key: 'a',
// desc: {value: 2, writable: true, enumerable: true, configurable: true}
new B();
```

为了处理这些目标，我们修补了 IC，使其在看到正在初始化的对象是代理时回退到运行时，如果对象上已经存在定义的字段，或者对象只是具有 IC 以前从未见过的隐藏类。如果边缘情况变得足够普遍，仍然可以优化它们，但到目前为止，为了实现的简单性而牺牲它们的性能似乎更好。

## 优化私有方法

### 私有方法的实现

在[规格](https://tc39.es/ecma262/#sec-privatemethodoraccessoradd)，则私有方法被描述为安装在实例上，但不安装在类上。但是，为了节省内存，V8 的实现将私有方法与私有品牌符号一起存储在与类关联的上下文中。调用构造函数时，V8 仅在实例中存储对该上下文的引用，并以自有品牌符号作为键。

![Evaluation and instantiation of classes with private methods](../_img/faster-class-features/class-evaluation-and-instantiation.svg)

当访问私有方法时，V8 从执行上下文开始遍历上下文链以查找类上下文，从找到的上下文中读取静态已知槽以获取类的私有品牌符号，然后检查实例是否具有由此品牌符号键入的属性，以查看实例是否从此类创建。如果品牌检查通过，V8 将从同一上下文中的另一个已知插槽加载私有方法并完成访问。

![Access of private methods](../_img/faster-class-features/access-private-methods.svg)

以以下代码片段为例：

```js
class A {
  #a() {}
}
```

V8 用于为 的构造函数生成以下字节码`A`:

```cpp
// Load the private brand symbol for class A from the context
// and store it into r1.
LdaImmutableCurrentContextSlot [3]
Star r1

// Load the target into r0.
Mov <this>, r0
// Load the current context into r2.
Mov <context>, r2
// Call the runtime %AddPrivateBrand() function to store the context in
// the instance with the private brand as key.
CallRuntime [AddPrivateBrand], r0-r2
```

因为还有对运行时函数的调用`%AddPrivateBrand()`，开销使构造函数比仅使用公共方法的类的构造函数慢得多。

### 优化自有品牌初始化

为了加快自有品牌的安装，在大多数情况下，我们只是重复使用`DefineKeyedOwnProperty`为优化私有字段而添加的机械：

```cpp
// Load the private brand symbol for class A from the context
// and store it into r1
LdaImmutableCurrentContextSlot [3]
Star0

// Use the DefineKeyedOwnProperty bytecode to store the
// context in the instance with the private brand as key
Ldar <context>
DefineKeyedOwnProperty <this>, r0, [0]
```

![Performance of instance initializations of classes with different methods](../_img/faster-class-features/private-methods-performance.svg)

但是，有一个警告：如果该类是其构造函数调用的派生类`super()`，私有方法的初始化 - 在我们的例子中，私有品牌符号的安装 - 必须在之后进行`super()`返回：

```js
class A {
  constructor() {
    // This throws from a new B() call because super() has not yet returned.
    this.callMethod();
  }
}

class B extends A {
  #method() {}
  callMethod() { return this.#method(); }
  constructor(o) {
    super();
  }
};
```

如前所述，在初始化品牌时，V8 还会在实例中存储对类上下文的引用。此引用不用于品牌检查，而是供调试器从实例中检索私有方法的列表，而无需知道它是从哪个类构造的。什么时候`super()`直接在构造函数中调用，V8 可以简单地从上下文寄存器加载上下文（这是`Mov <context>, r2`或`Ldar <context>`在上面的字节码中确实）来执行初始化，但是`super()`也可以从嵌套的箭头函数调用，而嵌套的箭头函数又可以从不同的上下文中调用。在本例中，V8 回退到运行时函数（仍名为`%AddPrivateBrand()`） 以在上下文链中查找类上下文，而不是依赖于上下文寄存器。例如，对于`callSuper`功能如下：

```js
class A extends class {} {
  #method() {}
  constructor(run) {
    const callSuper = () => super();
    // ...do something
    run(callSuper)
  }
};

new A((fn) => fn());
```

V8 现在生成以下字节码：

```cpp
// Invoke the super constructor to construct the instance
// and store it into r3.
...

// Load the private brand symbol from the class context at
// depth 1 from the current context and store it into r4
LdaImmutableContextSlot <context>, [3], [1]
Star4

// Load the depth 1 as an Smi into r6
LdaSmi [1]
Star6

// Load the current context into r5
Mov <context>, r5

// Use the %AddPrivateBrand() to locate the class context at
// depth 1 from the current context and store it in the instance
// with the private brand symbol as key
CallRuntime [AddPrivateBrand], r3-r6
```

在这种情况下，运行时调用的成本又回来了，因此与仅使用公共方法初始化类的实例相比，初始化此类的实例仍然会变慢。可以使用专用的字节码来实现什么`%AddPrivateBrand()`确实如此，但自调用以来`super()`在嵌套箭头函数中是相当罕见的，我们再次牺牲了性能来换取实现的简单性。

## 结语

这篇博客文章中提到的工作也包含在[节点.js 18.0.0 发布](https://nodejs.org/en/blog/announcements/v18-release-announce/).以前，Node.js在一些一直使用私有字段的内置类中切换到符号属性，以便将它们包含在嵌入式引导快照中并提高构造函数的性能（请参阅[这篇博客文章](https://www.nearform.com/blog/node-js-and-the-struggles-of-being-an-eventtarget/)以获取更多上下文）。随着 V8 中类功能的改进支持，Node.js[切换回私有类字段](https://github.com/nodejs/node/pull/42361)在这些类和 Node.js 的基准测试显示[这些更改未引入任何性能回归](https://github.com/nodejs/node/pull/42361#issuecomment-1068961385).

感谢 Igalia 和 Bloomberg 对此实现的贡献！
