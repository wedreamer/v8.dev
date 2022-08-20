***

标题： '`Promise.prototype.finally`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias))'
化身：

*   'mathias-bynens'
    日期： 2017-10-23
    标签：
*   ECMAScript
*   ES2018
    描述： 'Promise.prototype.终于允许在确定承诺时调用注册回调（即要么实现，要么被拒绝）。
    推文：“922459978857824261”

***

`Promise.prototype.finally`允许注册回调，以便在承诺*安定*（即要么已履行，要么被拒绝）。

假设您要获取一些要在页面上显示的数据。哦，您希望在请求开始时显示加载微调器，并在请求完成时隐藏它。当出现问题时，您会改为显示错误消息。

```js
const fetchAndDisplay = ({ url, element }) => {
  showLoadingSpinner();
  fetch(url)
    .then((response) => response.text())
    .then((text) => {
      element.textContent = text;
      hideLoadingSpinner();
    })
    .catch((error) => {
      element.textContent = error.message;
      hideLoadingSpinner();
    });
};

fetchAndDisplay({
  url: someUrl,
  element: document.querySelector('#output')
});
```

如果请求成功，我们将显示数据。如果出现问题，我们会改为显示错误消息。

在任何一种情况下，我们都需要调用`hideLoadingSpinner()`.到目前为止，我们别无选择，只能在两个`then()`和`catch()`块。跟`Promise.prototype.finally`，我们可以做得更好：

```js
const fetchAndDisplay = ({ url, element }) => {
  showLoadingSpinner();
  fetch(url)
    .then((response) => response.text())
    .then((text) => {
      element.textContent = text;
    })
    .catch((error) => {
      element.textContent = error.message;
    })
    .finally(() => {
      hideLoadingSpinner();
    });
};
```

这不仅可以减少代码重复，还可以更清楚地将成功/错误处理阶段和清理阶段分开。整洁！

目前，同样的事情是可能的`async`/`await`，和不带`Promise.prototype.finally`:

```js
const fetchAndDisplay = async ({ url, element }) => {
  showLoadingSpinner();
  try {
    const response = await fetch(url);
    const text = await response.text();
    element.textContent = text;
  } catch (error) {
    element.textContent = error.message;
  } finally {
    hideLoadingSpinner();
  }
};
```

因为[`async`和`await`严格来说更好](https://mathiasbynens.be/notes/async-stack-traces)，我们的建议仍然是使用它们而不是香草承诺。也就是说，如果您出于某种原因更喜欢香草承诺，`Promise.prototype.finally`可以帮助使你的代码更简单，更干净。

## `Promise.prototype.finally`支持 { #support }

<feature-support chrome="63 /blog/v8-release-63"
              firefox="58"
              safari="11.1"
              nodejs="10"
              babel="yes https://github.com/zloirock/core-js#ecmascript-promise"></feature-support>
