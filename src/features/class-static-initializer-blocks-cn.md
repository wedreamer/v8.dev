***

标题： “类静态初始化块”
作者： '郭淑宇 （[@\_shu](https://twitter.com/\_shu))'
化身：

*   “舒玉国”
    日期： 2021-03-30
    标签：
*   ECMAScript
    描述：'JavaScript类获得用于静态初始化的专用语法。
    推文：“1376925666780798989”

***

新的类静态初始化块语法允许开发人员收集应为给定类定义运行一次的代码，并将它们放在一个位置。请考虑以下示例，其中伪随机数生成器使用静态块初始化熵池一次，当`class MyPRNG`计算定义。

```js
class MyPRNG {
  constructor(seed) {
    if (seed === undefined) {
      if (MyPRNG.entropyPool.length === 0) {
        throw new Error('Entropy pool exhausted');
      }
      seed = MyPRNG.entropyPool.pop();
    }
    this.seed = seed;
  }

  getRandom() { … }

  static entropyPool = [];
  static {
    for (let i = 0; i < 512; i++) {
      this.entropyPool.push(probeEntropySource());
    }
  }
}
```

## 范围

每个静态初始化块都是自己的`var`和`let`/`const`范围。与静态字段初始值设定项一样，`this`静态块中的值是类构造函数本身。同样地`super.property`在静态块中引用超类的静态属性。

```js
var y = 'outer y';
class A {
  static fieldA = 'A.fieldA';
}
class B extends A {
  static fieldB = 'B.fieldB';
  static {
    let x = super.fieldA;
    // → 'A.fieldA'
    var y = this.fieldB;
    // → 'B.fieldB'
  }
}
// Since static blocks are their own `var` scope, `var`s do not hoist!
y;
// → 'outer y'
```

## 多个块

一个类可以有多个静态初始化块。这些块按文本顺序进行评估。此外，如果存在任何静态字段，则按文本顺序计算所有静态元素。

```js
class C {
  static field1 = console.log('field 1');
  static {
    console.log('static block 1');
  }
  static field2 = console.log('field 2');
  static {
    console.log('static block 2');
  }
}
// → field 1
//   static block 1
//   field 2
//   static block 2
```

## 访问私有字段

由于类静态初始化块始终嵌套在类中，因此它有权访问该类的私有字段。

```js
let getDPrivateField;
class D {
  #privateField;
  constructor(v) {
    this.#privateField = v;
  }
  static {
    getDPrivateField = (d) => d.#privateField;
  }
}
getDPrivateField(new D('private'));
// → private
```

仅此而已。快乐的面向对象！

## 类静态初始化块支持 { #support }

<feature-support chrome="91 https://bugs.chromium.org/p/v8/issues/detail?id=11375"
              firefox="no"
              safari="no"
              nodejs="no"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-class-static-block"></feature-support>
