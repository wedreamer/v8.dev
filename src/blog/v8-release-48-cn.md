***

标题： 'V8 发布 v4.8'
作者： 'V8团队'
日期： 2015-11-25 13：33：37
标签：

*   释放
    描述：“V8 v4.8 增加了对几个新的 ES2015 语言功能的支持。

***

大约每六周，我们创建一个新的 V8 分支，作为我们[发布流程](/docs/release-process).每个版本都是从V8的Git主节点分支出来的，紧接在Chrome分支之前，用于Chrome Beta里程碑。今天，我们很高兴地宣布我们最新的分行，[V8 版本 4.8](https://chromium.googlesource.com/v8/v8.git/+log/branch-heads/4.8)，它将处于测试阶段，直到与Chrome 48 Stable协同发布。V8 4.8 包含一些面向开发人员的功能，因此我们想在几周内预览一些亮点，以备发布。

## 改进的 ECMAScript 2015 （ES6） 支持

此版本的 V8 提供了对两个[知名符号](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Well-known_symbols)，来自ES2015规范的内置符号，允许开发人员利用以前隐藏的几个低级语言结构。

### `@@isConcatSpreadable`

布尔值属性的名称，如果`true`指示对象应通过以下方式平展到其数组元素`Array.prototype.concat`.

```js
(function() {
  'use strict';
  class AutomaticallySpreadingArray extends Array {
    get [Symbol.isConcatSpreadable]() {
      return true;
    }
  }
  const first = [1];
  const second = new AutomaticallySpreadingArray();
  second[0] = 2;
  second[1] = 3;
  const all = first.concat(second);
  // Outputs [1, 2, 3]
  console.log(all);
}());
```

### `@@toPrimitive`

要在对象上调用以隐式转换为基元值的方法的名称。

```js
(function(){
  'use strict';
  class V8 {
    [Symbol.toPrimitive](hint) {
      if (hint === 'string') {
        console.log('string');
        return 'V8';
      } else if (hint === 'number') {
        console.log('number');
        return 8;
      } else {
        console.log('default:' + hint);
        return 8;
      }
    }
  }

  const engine = new V8();
  console.log(Number(engine));
  console.log(String(engine));
}());
```

### `ToLength`

ES2015 规范调整了类型转换的抽象操作，以将参数转换为适合用作类似数组对象长度的整数。（虽然不能直接观察到，但在处理具有负长度的类似数组的对象时，此更改可能是间接可见的。

## V8 接口

请查看我们的[API 更改摘要](https://docs.google.com/document/d/1g8JFi8T_oAE\_7uAri7Njtig7fKaPDfotU6huOa1alds/edit).本文档在每个主要版本发布几周后定期更新。

具有[活动 V8 结账](https://v8.dev/docs/source-code#using-git)可以使用`git checkout -b 4.8 -t branch-heads/4.8`以试验 V8 v4.8 中的新功能。或者，您可以[订阅 Chrome 的 Beta 版频道](https://www.google.com/chrome/browser/beta.html)并很快自己尝试新功能。
