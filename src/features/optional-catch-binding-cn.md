***

标题：“可选`catch`绑定'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2018-03-27
    标签：
*   ECMAScript
*   ES2019
    描述： '在ES2019中，现在可以在没有参数的情况下使用捕获。
    推文：“956209997808939008”

***

这`catch`的子句`try`用于需要绑定的语句：

```js
try {
  doSomethingThatMightThrow();
} catch (exception) {
  //     ^^^^^^^^^
  // We must name the binding, even if we don’t use it!
  handleException();
}
```

在ES2019中，`catch`现在可以[在没有绑定的情况下使用](https://tc39.es/proposal-optional-catch-binding/).如果您不需要`exception`对象在处理异常的代码中。

```js
try {
  doSomethingThatMightThrow();
} catch { // → No binding!
  handleException();
}
```

## 自选`catch`绑定支持 { #support }

<feature-support chrome="66 /blog/v8-release-66#optional-catch-binding"
              firefox="58 https://bugzilla.mozilla.org/show_bug.cgi?id=1380881"
              safari="yes https://trac.webkit.org/changeset/220068/webkit"
              nodejs="10 https://github.com/nodejs/node/blob/master/doc/changelogs/CHANGELOG_V10.md#2018-04-24-version-1000-current-jasnell"
              babel="yes"></feature-support>
