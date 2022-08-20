***

标题：“公共和私有类字段”
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-12-13
    标签：
*   ECMAScript
*   ES2022
*   io19
*   节点.js 14
    描述：“一些提案通过新功能扩展了现有的JavaScript类语法。本文介绍了 V8 v7.2 和 Chrome 72 中新的公共类字段语法，以及即将推出的私有类字段语法。
    推文：“1121395767170740225”

***

一些提案通过新功能扩展了现有的JavaScript类语法。本文介绍了 V8 v7.2 和 Chrome 72 中新的公共类字段语法，以及即将推出的私有类字段语法。

下面是一个代码示例，用于创建名为`IncreasingCounter`:

```js
const counter = new IncreasingCounter();
counter.value;
// logs 'Getting the current value!'
// → 0
counter.increment();
counter.value;
// logs 'Getting the current value!'
// → 1
```

请注意，访问`value`在返回结果之前执行一些代码（即，它记录一条消息）。现在问问你自己，你会如何在JavaScript中实现这个类？🤔

## ES2015 类语法

操作方法如下`IncreasingCounter`可以使用 ES2015 类语法实现：

```js
class IncreasingCounter {
  constructor() {
    this._count = 0;
  }
  get value() {
    console.log('Getting the current value!');
    return this._count;
  }
  increment() {
    this._count++;
  }
}
```

该类将`value`getter 和 a`increment`原型上的方法。更有趣的是，该类具有一个构造函数，用于创建实例属性。`_count`并将其默认值设置为`0`.我们目前倾向于使用下划线前缀来表示`_count`不应该由该类的消费者直接使用，但这只是一个惯例;它不是*真*具有由语言强制执行的特殊语义的“私有”属性。

```js
const counter = new IncreasingCounter();
counter.value;
// logs 'Getting the current value!'
// → 0

// Nothing stops people from reading or messing with the
// `_count` instance property. 😢
counter._count;
// → 0
counter._count = 42;
counter.value;
// logs 'Getting the current value!'
// → 42
```

## 公共类字段

新的公共类字段语法允许我们简化类定义：

```js
class IncreasingCounter {
  _count = 0;
  get value() {
    console.log('Getting the current value!');
    return this._count;
  }
  increment() {
    this._count++;
  }
}
```

这`_count`属性现在很好地声明在类的顶部。我们不再需要一个构造函数来定义一些字段。整洁！

但是，`_count`字段仍然是公共属性。在此特定示例中，我们希望阻止用户直接访问该属性。

## 私有类字段

这就是私人类字段的用武之地。新的私有字段语法类似于公共字段，除了[您可以使用以下命令将字段标记为私有`#`](https://github.com/tc39/proposal-class-fields/blob/master/PRIVATE_SYNTAX_FAQ.md).您可以想到`#`作为字段名称的一部分：

```js
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Getting the current value!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}
```

私有字段在类体外部不可访问：

```js
const counter = new IncreasingCounter();
counter.#count;
// → SyntaxError
counter.#count = 42;
// → SyntaxError
```

## 公共和私有静态属性

类字段语法也可用于创建公共和私有静态属性和方法：

```js
class FakeMath {
  // `PI` is a static public property.
  static PI = 22 / 7; // Close enough.

  // `#totallyRandomNumber` is a static private property.
  static #totallyRandomNumber = 4;

  // `#computeRandomNumber` is a static private method.
  static #computeRandomNumber() {
    return FakeMath.#totallyRandomNumber;
  }

  // `random` is a static public method (ES2015 syntax)
  // that consumes `#computeRandomNumber`.
  static random() {
    console.log('I heard you like random numbers…');
    return FakeMath.#computeRandomNumber();
  }
}

FakeMath.PI;
// → 3.142857142857143
FakeMath.random();
// logs 'I heard you like random numbers…'
// → 4
FakeMath.#totallyRandomNumber;
// → SyntaxError
FakeMath.#computeRandomNumber();
// → SyntaxError
```

## 更简单的子类化

在处理引入其他字段的子类时，类字段语法的好处变得更加明显。想象一下以下基类`Animal`:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}
```

要创建`Cat`引入其他实例属性的子类，您以前必须调用`super()`以运行`Animal`创建属性之前的基类：

```js
class Cat extends Animal {
  constructor(name) {
    super(name);
    this.likesBaths = false;
  }
  meow() {
    console.log('Meow!');
  }
}
```

这是很多样板，只是为了表明猫不喜欢洗澡。幸运的是，类字段语法消除了对整个构造函数的需求，包括尴尬的`super()`叫：

```js
class Cat extends Animal {
  likesBaths = false;
  meow() {
    console.log('Meow!');
  }
}
```

## 功能支持 { #support }

### 支持公共类字段 { #support公共字段 }

<feature-support chrome="72 /blog/v8-release-72#public-class-fields"
              firefox="yes https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/69#JavaScript"
              safari="yes https://bugs.webkit.org/show_bug.cgi?id=174212"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-class-properties"></feature-support>

### 支持私有类字段 { #support私有字段 }

<feature-support chrome="74 /blog/v8-release-74#private-class-fields"
              firefox="90 https://spidermonkey.dev/blog/2021/05/03/private-fields-ship.html"
              safari="yes"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-class-properties"></feature-support>

### 支持私有方法和访问器 { #support私有方法 }

<feature-support chrome="84 /blog/v8-release-84#private-methods-and-accessors"
              firefox="90 https://spidermonkey.dev/blog/2021/05/03/private-fields-ship.html"
              safari="yes https://webkit.org/blog/11989/new-webkit-features-in-safari-15/"
              nodejs="14.6.0"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-private-methods"></feature-support>
