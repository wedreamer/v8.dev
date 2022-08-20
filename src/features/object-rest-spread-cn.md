***

标题： “对象静止和跨页属性”
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2017-06-06
    标签：
*   ECMAScript
*   ES2018
    描述： “本文解释了对象 rest 和 spread 属性在 JavaScript 中是如何工作的，并重新访问了数组 rest 和 spread 元素。
    推文：“890269994688315394”

***

讨论前*对象静止和跨页属性*，让我们沿着记忆小路走一趟，提醒自己一个非常相似的功能。

## ES2015 数组静止和传播元素 { #array-休息-传播 }

Good ol' ECMAScript 2015 推出*休息元素*用于数组解构赋值和*展开元素*用于数组文本。

```js
// Rest elements for array destructuring assignment:
const primes = [2, 3, 5, 7, 11];
const [first, second, ...rest] = primes;
console.log(first); // 2
console.log(second); // 3
console.log(rest); // [5, 7, 11]

// Spread elements for array literals:
const primesCopy = [first, second, ...rest];
console.log(primesCopy); // [2, 3, 5, 7, 11]
```

<feature-support chrome="47"
              firefox="16"
              safari="8"
              nodejs="6"
              babel="yes"></feature-support>

## ES2018：对象静止和传播属性 🆕 { #object-休息-传播 }

那么，有什么新内容呢？井[提案](https://github.com/tc39/proposal-object-rest-spread)还为对象文本启用 rest 和展开属性。

```js
// Rest properties for object destructuring assignment:
const person = {
    firstName: 'Sebastian',
    lastName: 'Markbåge',
    country: 'USA',
    state: 'CA',
};
const { firstName, lastName, ...rest } = person;
console.log(firstName); // Sebastian
console.log(lastName); // Markbåge
console.log(rest); // { country: 'USA', state: 'CA' }

// Spread properties for object literals:
const personCopy = { firstName, lastName, ...rest };
console.log(personCopy);
// { firstName: 'Sebastian', lastName: 'Markbåge', country: 'USA', state: 'CA' }
```

跨度属性提供了一种更优雅的替代方案[`Object.assign()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)在许多情况下：

```js
// Shallow-clone an object:
const data = { x: 42, y: 27, label: 'Treasure' };
// The old way:
const clone1 = Object.assign({}, data);
// The new way:
const clone2 = { ...data };
// Either results in:
// { x: 42, y: 27, label: 'Treasure' }

// Merge two objects:
const defaultSettings = { logWarnings: false, logErrors: false };
const userSettings = { logErrors: true };
// The old way:
const settings1 = Object.assign({}, defaultSettings, userSettings);
// The new way:
const settings2 = { ...defaultSettings, ...userSettings };
// Either results in:
// { logWarnings: false, logErrors: true }
```

但是，在传播处理设置器的方式上存在一些细微的差异：

1.  `Object.assign()`触发设定器;传播不会。
2.  您可以停止`Object.assign()`通过继承的只读属性创建自己的属性，而不是通过展开运算符。

[阿克塞尔·劳施迈尔的著作](http://2ality.com/2016/10/rest-spread-properties.html#spread-defines-properties-objectassign-sets-them)更详细地解释了这些陷阱。

<feature-support chrome="60"
              firefox="55"
              safari="11.1"
              nodejs="8.6"
              babel="yes"></feature-support>
