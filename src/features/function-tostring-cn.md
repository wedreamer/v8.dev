***

标题：“已修订`Function.prototype.toString`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-03-25
    标签：
*   ECMAScript
*   ES2019
    描述：'Function.prototype.toString 现在返回源代码文本的确切切片，包括空格和注释。

***

[`Function.prototype.toString()`](https://tc39.es/Function-prototype-toString-revision/)现在返回源代码文本的确切切片，包括空格和注释。下面是比较新旧行为的示例：

```js
// Note the comment between the `function` keyword
// and the function name, as well as the space following
// the function name.
function /* a comment */ foo () {}

// Previously, in V8:
foo.toString();
// → 'function foo() {}'
//             ^ no comment
//                ^ no space

// Now:
foo.toString();
// → 'function /* comment */ foo () {}'
```

## 功能支持 { #support }

<feature-support chrome="66 /blog/v8-release-66#function-tostring"
              firefox="yes"
              safari="no"
              nodejs="8"
              babel="no"></feature-support>
