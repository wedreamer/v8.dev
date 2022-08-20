***

标题： '`Symbol.prototype.description`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2019-06-25
    标签：
*   ECMAScript
*   ES2019
    描述： 'Symbol.prototype.description 提供了一种符合人体工程学的方法来访问符号的描述。
    推文：“1143432835665211394”

***

JavaScript`Symbol`s 可以在创建时给出一个描述：

```js
const symbol = Symbol('foo');
//                    ^^^^^
```

以前，以编程方式访问此描述的唯一方法是通过间接`Symbol.prototype.toString()`:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.toString();
// → 'Symbol(foo)'
//           ^^^
symbol.toString().slice(7, -1); // 🤔
// → 'foo'
```

但是，代码看起来有点神奇，不是很不言自明，并且违反了“明示意图，而不是实现”原则。上述技术也不允许您区分没有描述的符号（即`Symbol()`） 和一个以空字符串作为其描述的符号（即`Symbol('')`).

[新`Symbol.prototype.description`吸气剂](https://tc39.es/ecma262/#sec-symbol.prototype.description)提供了一种更符合人体工程学的方式来访问描述`Symbol`:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.description;
// → 'foo'
```

为`Symbol`s 没有描述，getter 返回`undefined`:

```js
const symbol = Symbol();
symbol.description;
// → undefined
```

## `Symbol.prototype.description`支持 { #support }

<feature-support chrome="70 /blog/v8-release-70#javascript-language-features"
              firefox="63"
              safari="12.1"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="yes https://github.com/zloirock/core-js#ecmascript-symbol"></feature-support>
