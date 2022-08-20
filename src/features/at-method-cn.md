***

标题： '`at`相对索引的方法'
作者： '郭淑宇 （[@\_shu](https://twitter.com/\_shu))'
化身：

*   “舒玉国”
    日期： 2021-07-13
    标签：
*   ECMAScript
    描述： 'JavaScript 现在有一个相对索引方法，用于数组、TypedArrays 和 Strings。

***

新`at`方法`Array.prototype`，各种TypedArray原型，以及`String.prototype`使访问更接近集合末尾的元素更容易、更简洁。

从集合的末尾访问第 N 个元素是一种常见操作。但是，通常的方法是冗长的，例如`my_array[my_array.length - N]`，或者可能不具有性能，例如`my_array.slice(-N)[0]`.新`at`通过将负指数解释为“从末端”来使此操作更加符合人体工程学。前面的例子可以表示为`my_array.at(-N)`.

为了统一性，还支持正指数，并且等效于普通属性访问。

这个新方法足够小，其完整的语义可以通过下面的合规polyfill实现来理解：

```js
function at(n) {
  // Convert the argument to an integer
  n = Math.trunc(n) || 0;
  // Allow negative indexing from the end
  if (n < 0) n += this.length;
  // Out-of-bounds access returns undefined
  if (n < 0 || n >= this.length) return undefined;
  // Otherwise, this is just normal property access
  return this[n];
}
```

## 关于字符串的一句话

因为`at`最终执行普通索引，调用`at`on String 值返回代码单位，就像普通索引一样。就像字符串上的普通索引一样，代码单元可能不是您想要的Unicode字符串！请考虑以下情况：[`String.prototype.codePointAt()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/codePointAt)更适合您的使用案例。

## `at`方法支持 { #support }

<feature-support chrome="92"
              firefox="90"
              safari="no"
              nodejs="no"
              babel="yes https://github.com/zloirock/core-js#relative-indexing-method"></feature-support>
