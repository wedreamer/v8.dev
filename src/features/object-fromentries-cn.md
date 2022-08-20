***

标题： '`Object.fromEntries`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)），JavaScript whisperer'
化身：

*   'mathias-bynens'
    日期： 2019-06-18
    标签：
*   ECMAScript
*   ES2019
*   io19
    描述： 'Object.fromEntries 是内置 JavaScript 库的有用补充，它补充了 Object.entries。
    推文：“1140993821897121796”

***

`Object.fromEntries`是内置 JavaScript 库的有用补充。在解释它的作用之前，它有助于理解预先存在的`Object.entries`应用程序接口。

## `Object.entries`

这`Object.entries`API已经存在了一段时间。

<feature-support chrome="54"
              firefox="47"
              safari="10.1"
              nodejs="7"
              babel="yes https://github.com/zloirock/core-js#ecmascript-object"></feature-support>

对于对象中的每个键值对，`Object.entries`给你一个数组，其中第一个元素是键，第二个元素是值。

`Object.entries`与 结合使用特别有用`for`-`of`，因为它使您能够非常优雅地迭代对象中的所有键值对：

```js
const object = { x: 42, y: 50 };
const entries = Object.entries(object);
// → [['x', 42], ['y', 50]]

for (const [key, value] of entries) {
  console.log(`The value of ${key} is ${value}.`);
}
// Logs:
// The value of x is 42.
// The value of y is 50.
```

不幸的是，没有简单的方法可以从条目结果返回到等效对象...直到现在！

## `Object.fromEntries`

新`Object.fromEntries`API 执行的反函数`Object.entries`.这使得根据对象的条目重建对象变得容易：

```js
const object = { x: 42, y: 50 };
const entries = Object.entries(object);
// → [['x', 42], ['y', 50]]

const result = Object.fromEntries(entries);
// → { x: 42, y: 50 }
```

一个常见的用例是转换对象。现在，您可以通过循环访问其条目，然后使用您可能已经熟悉的数组方法来执行此操作：

```js
const object = { x: 42, y: 50, abc: 9001 };
const result = Object.fromEntries(
  Object.entries(object)
    .filter(([ key, value ]) => key.length === 1)
    .map(([ key, value ]) => [ key, value * 2 ])
);
// → { x: 84, y: 100 }
```

在这个例子中，我们`filter`将对象设置为仅获取长度的键`1`，即仅按键`x`和`y`，但不是密钥`abc`.然后我们`map`在其余条目上，并为每个条目返回更新的键值对。在此示例中，我们将每个值乘以`2`.最终结果是一个新对象，仅具有属性`x`和`y`和新值。

## 对象与地图

JavaScript 还支持`Map`s，它们通常是比常规对象更合适的数据结构。因此，在您完全控制的代码中，您可能使用的是映射而不是对象。但是，作为开发人员，您并不总是可以选择表示形式。有时，您正在操作的数据来自外部 API 或某个库函数，该函数为您提供对象而不是映射。

`Object.entries`使将对象转换为地图变得容易：

```js
const object = { language: 'JavaScript', coolness: 9001 };

// Convert the object into a map:
const map = new Map(Object.entries(object));
```

反之亦然：即使您的代码使用映射，也可能需要在某个时候序列化数据，例如将其转换为 JSON 以发送 API 请求。或者，也许您需要将数据传递到另一个需要对象而不是映射的库。在这些情况下，您需要基于地图数据创建对象。`Object.fromEntries`使这变得微不足道：

```js
// Convert the map back into an object:
const objectCopy = Object.fromEntries(map);
// → { language: 'JavaScript', coolness: 9001 }
```

两者兼而有之`Object.entries`和`Object.fromEntries`在语言中，您现在可以轻松地在地图和对象之间进行转换。

### 警告：当心数据丢失 { #data丢失 }

在将映射转换为纯对象（如上例所示）时，有一个隐式假设，即每个键都唯一地字符串化。如果此假设不成立，则会发生数据丢失：

```js
const map = new Map([
  [{}, 'a'],
  [{}, 'b'],
]);
Object.fromEntries(map);
// → { '[object Object]': 'b' }
// Note: the value 'a' is nowhere to be found, since both keys
// stringify to the same value of '[object Object]'.
```

使用前`Object.fromEntries`或任何其他将地图转换为对象的技术，请确保地图的键产生唯一`toString`结果。

## `Object.fromEntries`支持 { #support }

<feature-support chrome="73 /blog/v8-release-73#object.fromentries"
              firefox="63"
              safari="12.1"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-object"></feature-support>
