***

标题：“错误原因”
作者： '维克多·戈麦斯 （[@VictorBFG](https://twitter.com/VictorBFG))'
化身：

*   “胜利者戈麦斯”
    日期： 2021-07-07
    标签：
*   ECMAScript
    描述： 'JavaScript 现在支持错误原因。
    推文：“1412774651558862850”

***

假设您有一个函数正在调用两个单独的工作负载`doSomeWork`和`doMoreWork`.这两个函数可以引发相同类型的错误，但您需要以不同的方式处理它们。

捕获错误并将其与其他上下文信息一起抛出是解决此问题的常用方法，例如：

```js
function doWork() {
  try {
    doSomeWork();
  } catch (err) {
    throw new CustomError('Some work failed', err);
  }
  doMoreWork();
}

try {
  doWork();
} catch (err) {
  // Is |err| coming from |doSomeWork| or |doMoreWork|?
}
```

不幸的是，上述解决方案很费力，因为需要创建自己的解决方案`CustomError`.而且，更糟糕的是，没有开发人员工具能够为意外异常提供有用的诊断消息，因为在如何正确表示这些错误方面没有达成共识。

到目前为止，一直缺少的是连锁错误的标准方法。JavaScript 现在支持错误原因。可以将其他选项参数添加到`Error`构造函数`cause`属性，其值将分配给错误实例。然后可以很容易地将错误链接起来。

```js
function doWork() {
  try {
    doSomeWork();
  } catch (err) {
    throw new Error('Some work failed', { cause: err });
  }
  try {
    doMoreWork();
  } catch (err) {
    throw new Error('More work failed', { cause: err });
  }
}

try {
  doWork();
} catch (err) {
  switch(err.message) {
    case 'Some work failed':
      handleSomeWorkFailure(err.cause);
      break;
    case 'More work failed':
      handleMoreWorkFailure(err.cause);
      break;
  }
}
```

此功能在 V8 v9.3 中可用。

## 错误导致支持 { #support }

<feature-support chrome="93 https://chromium-review.googlesource.com/c/v8/v8/+/2784681"
              firefox="91 https://bugzilla.mozilla.org/show_bug.cgi?id=1679653"
              safari="15 https://bugs.webkit.org/show_bug.cgi?id=223302"
              nodejs="no"
              babel="no"></feature-support>
