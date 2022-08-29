***

标题： “高性能ES2015及以后”
作者： '贝内迪克特·穆勒[@bmeurer](https://twitter.com/bmeurer)， ECMAScript 性能工程师
化身：

*   'benedikt-meurer'
    日期： 2017-02-17 13：33：37
    标签：
*   ECMAScript
    描述： “V8 的 ES2015+ 语言功能性能现在与转译的 ES5 版本相当。

***

在过去的几个月里，V8团队专注于带来新添加的性能。[ES2015](https://www.ecma-international.org/ecma-262/6.0/)以及其他更新的 JavaScript 功能，可与它们转译的功能相媲美[夏令时](https://www.ecma-international.org/ecma-262/5.1/)同行。

## 动机

在详细介绍各种改进之前，我们应该首先考虑为什么ES2015 +功能的性能很重要，尽管广泛使用[巴别塔](http://babeljs.io/)在现代网络开发中：

1.  首先，有新的ES2015功能仅按需进行多边形填充，例如[`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)内置。当巴别塔转译时[对象展开属性](https://github.com/sebmarkbage/ecmascript-rest-spread)（被许多人大量使用[反应](https://facebook.github.io/react)和[Redux](http://redux.js.org/)应用程序），它依赖于[`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)而不是 ES5 等效项（如果 VM 支持）。
2.  Polyfill ES2015 功能通常会增加代码大小，这对当前[网络性能危机](https://channel9.msdn.com/Blogs/msedgedev/nolanlaw-web-perf-crisis)，特别是在新兴市场常见的移动设备上。因此，仅仅交付、解析和编译代码的成本可能相当高，甚至在你达到实际执行成本之前。
3.  最后但并非最不重要的一点是，客户端JavaScript只是依赖于V8引擎的环境之一。还有[节点.js](https://nodejs.org/)对于服务器端应用程序和工具，开发人员不需要转译为ES5代码，但可以直接使用[相关 V8 版本](https://nodejs.org/en/download/releases/)在目标节点.js发布。

让我们考虑以下代码片段[Redux 文档](http://redux.js.org/docs/recipes/UsingObjectSpreadOperator.html):

```js
function todoApp(state = initialState, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return { ...state, visibilityFilter: action.filter };
    default:
      return state;
  }
}
```

该代码中有两个需要转译的内容：状态的默认参数和将状态扩展到对象文本中。Babel 生成以下 ES5 代码：

```js
'use strict';

var _extends = Object.assign || function(target) {
  for (var i = 1; i < arguments.length; i++) {
    var source = arguments[i];
    for (var key in source) {
      if (Object.prototype.hasOwnProperty.call(source, key)) {
        target[key] = source[key];
      }
    }
  }
  return target;
};

function todoApp() {
  var state = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : initialState;
  var action = arguments[1];

  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return _extends({}, state, { visibilityFilter: action.filter });
    default:
      return state;
  }
}
```

现在想象一下[`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)比多填充慢几个数量级`_extends`由巴别塔生成。在这种情况下，从不支持的浏览器升级[`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)对于支持ES2015的浏览器版本将是一个严重的性能下降，并且可能会阻碍ES2015的采用。

此示例还强调了转译的另一个重要缺点：提供给用户的生成代码通常比开发人员最初编写的 ES2015+ 代码大得多。在上面的示例中，原始代码为 203 个字符（176 字节 gzip），而生成的代码为 588 个字符（367 字节 gzip）。这已经是尺寸增加的两倍。让我们看一下另一个示例[异步迭代器](https://github.com/tc39/proposal-async-iteration)建议：

```js
async function* readLines(path) {
  let file = await fileOpen(path);
  try {
    while (!file.EOF) {
      yield await file.readLine();
    }
  } finally {
    await file.close();
  }
}
```

Babel 将这 187 个字符（150 字节 gzpped）转换为 ES5 代码的 2987 个字符（971 字节 gzpped），甚至不算[再生器运行时](https://babeljs.io/docs/plugins/transform-regenerator/)作为附加依赖项需要：

```js
'use strict';

var _asyncGenerator = function() {
  function AwaitValue(value) {
    this.value = value;
  }

  function AsyncGenerator(gen) {
    var front, back;

    function send(key, arg) {
      return new Promise(function(resolve, reject) {
        var request = {
          key: key,
          arg: arg,
          resolve: resolve,
          reject: reject,
          next: null
        };
        if (back) {
          back = back.next = request;
        } else {
          front = back = request;
          resume(key, arg);
        }
      });
    }

    function resume(key, arg) {
      try {
        var result = gen[key](arg);
        var value = result.value;
        if (value instanceof AwaitValue) {
          Promise.resolve(value.value).then(function(arg) {
            resume('next', arg);
          }, function(arg) {
            resume('throw', arg);
          });
        } else {
          settle(result.done ? 'return' : 'normal', result.value);
        }
      } catch (err) {
        settle('throw', err);
      }
    }

    function settle(type, value) {
      switch (type) {
        case 'return':
          front.resolve({
            value: value,
            done: true
          });
          break;
        case 'throw':
          front.reject(value);
          break;
        default:
          front.resolve({
            value: value,
            done: false
          });
          break;
      }
      front = front.next;
      if (front) {
        resume(front.key, front.arg);
      } else {
        back = null;
      }
    }
    this._invoke = send;
    if (typeof gen.return !== 'function') {
      this.return = undefined;
    }
  }
  if (typeof Symbol === 'function' && Symbol.asyncIterator) {
    AsyncGenerator.prototype[Symbol.asyncIterator] = function() {
      return this;
    };
  }
  AsyncGenerator.prototype.next = function(arg) {
    return this._invoke('next', arg);
  };
  AsyncGenerator.prototype.throw = function(arg) {
    return this._invoke('throw', arg);
  };
  AsyncGenerator.prototype.return = function(arg) {
    return this._invoke('return', arg);
  };
  return {
    wrap: function wrap(fn) {
      return function() {
        return new AsyncGenerator(fn.apply(this, arguments));
      };
    },
    await: function await (value) {
      return new AwaitValue(value);
    }
  };
}();

var readLines = function () {
  var _ref = _asyncGenerator.wrap(regeneratorRuntime.mark(function _callee(path) {
    var file;
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            _context.next = 2;
            return _asyncGenerator.await(fileOpen(path));

          case 2:
            file = _context.sent;
            _context.prev = 3;

          case 4:
            if (file.EOF) {
              _context.next = 11;
              break;
            }

            _context.next = 7;
            return _asyncGenerator.await(file.readLine());

          case 7:
            _context.next = 9;
            return _context.sent;

          case 9:
            _context.next = 4;
            break;

          case 11:
            _context.prev = 11;
            _context.next = 14;
            return _asyncGenerator.await(file.close());

          case 14:
            return _context.finish(11);

          case 15:
          case 'end':
            return _context.stop();
        }
      }
    }, _callee, this, [[3,, 11, 15]]);
  }));

  return function readLines(_x) {
    return _ref.apply(this, arguments);
  };
}();
```

这是一个**650%**增加大小（通用`_asyncGenerator`函数可能是可共享的，具体取决于您捆绑代码的方式，因此您可以在异步迭代器的多个用途中分摊部分成本）。我们认为长期只提供转译到ES5的代码是不可行的，因为大小的增加不仅会影响下载时间/成本，还会增加解析和编译的额外开销。如果我们真的想大幅提高现代Web应用程序的页面加载和敏捷性，特别是在移动设备上，我们必须鼓励开发人员在编写代码时不仅要使用ES2015 +，还要将其交付给ES5，而不是转译到ES5。仅将完全转译的捆绑包交付给不支持 ES2015 的旧版浏览器。对于虚拟机实施者来说，这一愿景意味着我们需要原生支持ES2015+功能。**和**提供合理的性能。

## 测量方法

如上所述，ES2015+功能的绝对性能在这一点上并不是一个真正的问题。相反，目前最优先考虑的是确保ES2015 +功能的性能与其朴素的ES5相当，更重要的是，与Babel生成的版本相当。方便的是，已经有一个名为[六速](https://github.com/kpdecker/six-speed)由[凯文·德克尔](http://www.incaseofstairs.com/)，这或多或少地完成了我们所需要的：ES2015 功能与朴素的 ES5 与转译器生成的代码的性能比较。

![The SixSpeed benchmark](../_img/high-performance-es2015/sixspeed.png)

因此，我们决定将其作为我们最初的ES2015 +性能工作的基础。我们[分叉六速](https://fhinkel.github.io/six-speed/)并添加了几个基准测试。我们首先关注最严重的回归，即从幼稚的ES5到推荐的ES2015 +版本的减速速度超过2倍的行项目，因为我们的基本假设是，幼稚的ES5版本将至少与Babel生成的符合规范的版本一样快。

## 现代语言的现代建筑

在过去，V8很难优化ES2015+中的语言特性。例如，将异常处理（即 try/catch/finally）支持添加到 V8 的经典优化编译器 Crankshaft 中从未变得可行。这意味着 V8 能够优化 ES6 功能，例如...of，它基本上有一个隐含的最后条款，是有限的。Crankshaft的局限性以及向全代码机（V8的基准编译器）添加新语言功能的整体复杂性，使得确保在V8中添加和优化新的ES功能与标准化一样快地变得非常困难。

幸运的是，点火和涡轮风扇（[V8 的新解释器和编译器管道](/blog/test-the-future)），旨在从一开始就支持整个JavaScript语言，包括高级控制流，异常处理和最近的`for`-`of`并从ES2015中解构。Ignition和TurboFan架构的紧密集成使得快速添加新功能并快速和增量地优化它们成为可能。

我们对现代语言功能的许多改进只有在新的Ignition/TurboFan管道下才可行。事实证明，点火和涡轮风扇对于优化发电机和异步功能尤其重要。发电机长期以来一直由V8支持，但由于曲轴的控制流量限制而无法优化。异步函数本质上是生成器之上的糖，因此它们属于同一类别。新的编译器管道利用Ignition来理解AST并生成字节码，这些字节码将复杂的生成器控制分解为更简单的本地控制流字节码。TurboFan可以更轻松地优化生成的字节码，因为它不需要知道有关发电机控制流程的任何具体信息，只需了解如何保存和恢复函数的收益率状态即可。

![How JavaScript generators are represented in Ignition and TurboFan](../_img/high-performance-es2015/generators.svg)

## 国情咨文

我们的短期目标是尽快达到平均减速2×以下。我们首先查看最差的测试，从Chrome 54到Chrome 58（金丝雀），我们设法将减速超过2×的测试数量从16个减少到8个，同时将Chrome 54中最糟糕的减速从19×减少到Chrome 58（金丝雀）的6×我们还显著降低了在此期间的平均值和中位数放缓：

![Slowdown of ES2015+ compared to native ES5 equivalent](../_img/high-performance-es2015/slowdown.svg)

您可以看到ES2015 +和ES5的奇偶校验的明显趋势。平均而言，与 ES5 相比，我们的性能提高了 47% 以上。以下是自 Chrome 54 以来我们解决的一些亮点。

![ES2015+ performance compared to naive ES5 equivalent](../_img/high-performance-es2015/comparison.svg)

最值得注意的是，我们改进了基于迭代的新语言结构的性能，如传播运算符，解构和`for`-`of`循环。例如，使用数组解构：

```js
function fn() {
  var [c] = data;
  return c;
}
```

...现在和幼稚的ES5版本一样快：

```js
function fn() {
  var c = data[0];
  return c;
}
```

...并且比 Babel 生成的代码更快（也更短）：

```js
'use strict';

var _slicedToArray = function() {
  function sliceIterator(arr, i) {
    var _arr = [];
    var _n = true;
    var _d = false;
    var _e = undefined;
    try {
      for (var _i = arr[Symbol.iterator](), _s; !(_n = (_s = _i.next()).done); _n = true) {
        _arr.push(_s.value);
        if (i && _arr.length === i) break;
      }
    } catch (err) {
      _d = true;
      _e = err;
    } finally {
      try {
        if (!_n && _i['return']) _i['return']();
      } finally {
        if (_d) throw _e;
      }
    }
    return _arr;
  }
  return function(arr, i) {
    if (Array.isArray(arr)) {
      return arr;
    } else if (Symbol.iterator in Object(arr)) {
      return sliceIterator(arr, i);
    } else {
      throw new TypeError('Invalid attempt to destructure non-iterable instance');
    }
  };
}();

function fn() {
  var _data = data,
      _data2 = _slicedToArray(_data, 1),
      c = _data2[0];

  return c;
}
```

您可以查看[高速ES2015](https://docs.google.com/presentation/d/1wiiZeRQp8-sXDB9xXBUAGbaQaWJC84M5RNxRyQuTmhk)我们最后一次演讲[慕尼黑 NodeJS 用户组](http://www.mnug.de/)聚会了解更多详情：

我们致力于继续提高ES2015+功能的性能。如果您对细节感兴趣，请查看V8的[ES2015 及更高性能计划](https://docs.google.com/document/d/1EA9EbfnydAmmU_lM8R_uEMQ-U_v4l9zulePSBkeYWmY).
