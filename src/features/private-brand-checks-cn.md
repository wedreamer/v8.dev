***

标题：“私人品牌支票又名。`#foo in obj`'
作者： 'Marja Hölttä （[@marjakh](https://twitter.com/marjakh))'
化身：

*   'marja-holtta'
    日期： 2021-04-14
    标签：
*   ECMAScript
    描述：“自有品牌检查允许测试对象中是否存在私有字段。
    推文：“1382327454975590401”

***

这[`in`算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in)可用于测试给定对象（或其原型链中的任何对象）是否具有给定属性：

```javascript
const o1 = {'foo': 0};
console.log('foo' in o1); // true
const o2 = {};
console.log('foo' in o2); // false
const o3 = Object.create(o1);
console.log('foo' in o3); // true
```

自有品牌检查功能扩展了`in`操作员支持[私有类字段](https://v8.dev/features/class-fields#private-class-fields):

```javascript
class A {
  static test(obj) {
    console.log(#foo in obj);
  }
  #foo = 0;
}

A.test(new A()); // true
A.test({}); // false

class B {
  #foo = 0;
}

A.test(new B()); // false; it's not the same #foo
```

由于私有名称仅在定义它们的类中可用，因此测试也必须在类内部进行，例如在类似`static test`以上。

子类实例从父类接收私有字段作为 own 属性：

```javascript
class SubA extends A {};
A.test(new SubA()); // true
```

但是使用`Object.create`（或者稍后通过`__proto__`二传手或`Object.setPrototypeOf`） 不接收私有字段作为 own 属性。由于私有字段查找仅适用于自己的属性，因此`in`运算符找不到以下继承的字段：

```javascript
const a = new A();
const o = Object.create(a);
A.test(o); // false, private field is inherited and not owned
A.test(o.__proto__); // true

const o2 = {};
Object.setPrototypeOf(o2, a);
A.test(o2); // false, private field is inherited and not owned
A.test(o2.__proto__); // true
```

访问不存在的私有字段会引发错误 - 与普通属性不同，在普通属性中，访问不存在的属性会返回`undefined`但不扔。在自有品牌检查之前，开发商已经被迫使用`try`-`catch`用于在对象没有所需私有字段的情况下实现回退行为：

```javascript
class D {
  use(obj) {
    try {
      obj.#foo;
    } catch {
      // Fallback for the case obj didn't have #foo
    }
  }
  #foo = 0;
}
```

现在，可以使用私有品牌检查来测试私有字段的存在：

```javascript
class E {
  use(obj) {
    if (#foo in obj) {
      obj.#foo;
    } else {
      // Fallback for the case obj didn't have #foo
    }
  }
  #foo = 0;
}
```

但要注意 - 一个私有字段的存在并不能保证该对象具有在类中声明的所有私有字段！下面的示例演示一个半构造对象，该对象在其类中声明的两个私有字段中只有一个：

```javascript
let halfConstructed;
class F {
  m() {
    console.log(#x in this); // true
    console.log(#y in this); // false
  }
  #x = 0;
  #y = (() => {
    halfConstructed = this;
    throw 'error';
  })();
}

try {
  new F();
} catch {}

halfConstructed.m();
```

## 自有品牌检查支持 { #support }

<功能支持 chrome=“91 https://bugs.chromium.org/p/v8/issues/detail?id=11374” firefox=“no” safari=“no” nodejs=“no” babel=“no”></feature-support>
