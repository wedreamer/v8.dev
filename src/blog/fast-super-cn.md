***

标题： '超快`super`物业访问'
作者： '[Marja Hölttä](https://twitter.com/marjakh)，超级优化器
化身：

*   马利亚-霍尔塔
    日期： 2021-02-18
    标签：
*   JavaScript
    描述： “V8 v9.0 中更快的超级属性访问”
    推文：“1362465295848333316”

***

这[`super`关键词](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)可用于访问对象父级上的属性和函数。

以前，访问超级属性（如`super.x`） 是通过运行时调用实现的。从 V8 v9.0 开始，我们重用[内联高速缓存 （IC） 系统](https://mathiasbynens.be/notes/shapes-ics)在未优化的代码中，并为超级属性访问生成适当的优化代码，而不必跳转到运行时。

从下图中可以看出，由于运行时调用，超级属性访问曾经比正常属性访问慢一个数量级。现在我们离平起平坐了更近的路。

![Compare super property access to regular property access, optimized](/\_img/fast-super/super-opt.svg)

![Compare super property access to regular property access, unoptimized](/\_img/fast-super/super-no-opt.svg)

超级属性访问很难进行基准测试，因为它必须发生在函数内部。我们无法对单个属性访问进行基准测试，而只能对更大的工作块进行基准测试。因此，函数调用开销包含在度量中。上图有点低估了超级属性访问和正常属性访问之间的差异，但它们足够准确，可以证明新旧超级属性访问之间的差异。

在未优化（解释）模式下，超级属性访问将始终比正常属性访问慢，因为我们需要执行更多加载（从上下文中读取主对象并读取`__proto__`从主对象）。在优化的代码中，我们已经尽可能将 home 对象作为常量嵌入。这可以通过嵌入其`__proto__`作为一个常量。

### 原型遗传和`super`

让我们从基础开始 - 超级财产访问到底意味着什么？

```javascript
class A { }
A.prototype.x = 100;

class B extends A {
  m() {
    return super.x;
  }
}
const b = new B();
b.m();
```

现在`A`是超类的`B`和`b.m()`返回`100`如您所料。

![Class inheritance diagram](/\_img/fast-super/inheritance-1.svg)

现实情况[JavaScript 的原型继承](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)更复杂：

![Prototypal inheritance diagram](/\_img/fast-super/inheritance-2.svg)

我们需要仔细区分`__proto__`和`prototype`属性 - 它们的含义不是一样的！为了使它更加混乱，对象`b.__proto__`通常被称为”`b`的原型”。

`b.__proto__`是对象，从`b`继承属性。`B.prototype`是对象，它将是`__proto__`使用`new B()`那是`b.__proto__ === B.prototype`.

挨次`B.prototype`有自己的`__proto__`等于的属性`A.prototype`.总之，这形成了所谓的原型链：

    b ->
     b.__proto__ === B.prototype ->
      B.prototype.__proto__ === A.prototype ->
       A.prototype.__proto__ === Object.prototype ->
        Object.prototype.__proto__ === null

通过这条链，`b`可以访问任何这些对象中定义的所有属性。方法`m`是 的属性`B.prototype`—`B.prototype.m`—— 这就是为什么`b.m()`工程。

现在我们可以定义`super.x`里面`m`作为属性查找，我们开始查找属性`x`在*家对象的* `__proto__`并沿着原型链向上走，直到我们找到它。

home 对象是定义方法的对象 - 在本例中为 home 对象`m`是`B.prototype`.其`__proto__`是`A.prototype`，这就是我们开始寻找房产的地方`x`.我们将致电`A.prototype`这*查找启动对象*.在这种情况下，我们找到属性`x`立即在查找启动对象中，但通常它也可能位于原型链的更上端。

如果`B.prototype`有一个属性称为`x`，我们会忽略它，因为我们开始在原型链中寻找它。此外，在这种情况下，超级属性查找不依赖于*接收器*- 对象即`this`值（调用方法时）。

```javascript
B.prototype.m.call(some_other_object); // still returns 100
```

如果属性有一个 getter，则接收方将作为 getter 传递给 getter`this`价值。

总结一下：在超级属性访问中，`super.x`，则查找起始对象为`__proto__`的主对象和接收器是发生超级属性访问的方法的接收器。

在正常属性访问中，`o.x`，我们开始寻找房产`x`在`o`并走上原型链。我们还将使用`o`作为接收者，如果`x`碰巧有一个 getter - 查找起始对象和接收器是同一个对象 （`o`).

*超级属性访问与常规属性访问类似，其中查找起始对象和接收器不同。*

### 实施速度更快`super`

上述实现也是实现快速超级属性访问的关键。V8 已经设计为使属性访问快速 - 现在我们针对接收器和查找启动对象不同的情况对其进行了推广。

V8 的数据驱动型内联缓存系统是实现快速属性访问的核心部分。您可以在[高层介绍](https://mathiasbynens.be/notes/shapes-ics)以上链接，或更详细的描述[V8 的对象表示](https://v8.dev/blog/fast-properties)和[V8 的数据驱动型内联缓存系统是如何实现的](https://docs.google.com/document/d/1mEhMn7dbaJv68lTAvzJRCQpImQoO6NZa61qRimVeA-k/edit?usp=sharing).

加快速度`super`，我们添加了一个新的[点火](https://v8.dev/docs/ignition)字节码，`LdaNamedPropertyFromSuper`，这使我们能够以解释模式插入IC系统，并为超级属性访问生成优化代码。

使用新的字节码，我们可以添加一个新的IC，`LoadSuperIC`，用于加快超级属性加载速度。似`LoadIC`它处理正常的属性载荷，`LoadSuperIC`跟踪它所看到的查找启动对象的形状，并记住如何从具有这些形状之一的对象加载属性。

`LoadSuperIC`将现有的 IC 机械重新用于属性负载，只是使用不同的查找启动对象。由于IC层已经区分了查找启动对象和接收器，因此实现应该很容易。但是，由于查找开始对象和接收器始终相同，因此存在一些错误，即使我们指的是接收器，我们也会使用查找启动对象，反之亦然。这些错误已得到修复，我们现在正确支持查找启动对象和接收器不同的情况。

超级属性访问的优化代码由`JSNativeContextSpecialization`阶段[涡轮风扇](https://v8.dev/docs/turbofan)编译器。实现概括现有的属性查找机制 （[`JSNativeContextSpecialization::ReduceNamedAccess`](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/compiler/js-native-context-specialization.cc;l=1130)） 来处理接收方和查找启动对象不同的情况。

当我们将主对象移出`JSFunction`存储位置。它现在存储在类上下文中，这使得TurboFan尽可能地将其作为常量嵌入到优化的代码中。

## 其他用法`super`

`super`inside object 文本方法的工作方式与类方法内部一样，并且进行了类似的优化。

```javascript
const myproto = {
  __proto__: { 'x': 100 },
  m() { return super.x; }
};
const o = { __proto__: myproto };
o.m(); // returns 100
```

当然，有些角落案例我们没有进行优化。例如，编写超级属性 （`super.x = ...`） 未优化。此外，使用mixins会使访问站点变得巨型，从而导致超级属性访问速度变慢：

```javascript
function createMixin(base) {
  class Mixin extends base {
    m() { return super.m() + 1; }
    //                ^ this access site is megamorphic
  }
  return Mixin;
}

class Base {
  m() { return 0; }
}

const myClass = createMixin(
  createMixin(
    createMixin(
      createMixin(
        createMixin(Base)
      )
    )
  )
);
(new myClass()).m();
```

要确保所有面向对象的模式尽可能快，仍有工作要做 - 请继续关注进一步的优化！
