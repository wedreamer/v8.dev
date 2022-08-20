***

标题： '`Object.hasOwn`'
作者： '维克多·戈麦斯 （[@VictorBFG](https://twitter.com/VictorBFG))'
化身：

*   “胜利者戈麦斯”
    日期： 2021-07-01
    标签：
*   ECMAScript
    描述： '`Object.hasOwn`使`Object.prototype.hasOwnProperty`更容易接近。
    推文：“1410577516943847424”

***

今天，编写这样的代码非常普遍：

```js
const hasOwnProperty = Object.prototype.hasOwnProperty;

if (hasOwnProperty.call(object, 'foo')) {
  // `object` has property `foo`.
}
```

或者使用公开简单版本的库`Object.prototype.hasOwnProperty`如[有](https://www.npmjs.com/package/has)或[lodash.has](https://www.npmjs.com/package/lodash.has).

随着[`Object.hasOwn`建议](https://github.com/tc39/proposal-accessible-object-hasownproperty)，我们可以简单地写：

```js
if (Object.hasOwn(object, 'foo')) {
  // `object` has property `foo`.
}
```

`Object.hasOwn`已在 V8 v9.3 后面可用`--harmony-object-has-own`标记，我们很快就会在Chrome中推出它。

## `Object.hasOwn`支持 { #support }

<feature-support chrome="yes https://chromium-review.googlesource.com/c/v8/v8/+/2922117"
              firefox="yes https://hg.mozilla.org/try/rev/94515f78324e83d4fd84f4b0ab764b34aabe6d80"
              safari="yes https://bugs.webkit.org/show_bug.cgi?id=226291"
              nodejs="no"
              babel="yes https://github.com/zloirock/core-js#accessible-objectprototypehasownproperty"></feature-support>
