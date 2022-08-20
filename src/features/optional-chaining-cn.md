***

标题： “可选链接”
作者： '玛雅·阿米拉诺娃 （[@Zmayski](https://twitter.com/Zmayski)），可选链条的断路器
化身：

*   '玛雅-军队'
    日期： 2019-08-27
    标签：
*   ECMAScript
*   ES2020
    描述：“可选链接通过内置的空检查实现属性访问的可读和简洁的表达式。
    推文：“1166360971914481669”

***

JavaScript 中的长属性访问链可能容易出错，因为它们中的任何一个都可能评估为`null`或`undefined`（也称为“空值”）。检查每个步骤的属性是否存在很容易变成一个深度嵌套的结构`if`-语句或长语句`if`-条件复制属性访问链：

```js
// Error prone-version, could throw.
const nameLength = db.user.name.length;

// Less error-prone, but harder to read.
let nameLength;
if (db && db.user && db.user.name)
  nameLength = db.user.name.length;
```

上述内容也可以使用三元运算符来表示，这并不完全有助于提高可读性：

```js
const nameLength =
  (db
    ? (db.user
      ? (db.user.name
        ? db.user.name.length
        : undefined)
      : undefined)
    : undefined);
```

## 介绍可选链接运算符 { #optional链接 }

当然，您不想编写这样的代码，因此最好有一些替代方案。其他一些语言通过使用称为“可选链接”的功能为这个问题提供了一个优雅的解决方案。根据[最近的规范提案](https://github.com/tc39/proposal-optional-chaining)，“可选链是一个或多个属性访问和函数调用的链，其中第一个以令牌开头`?.`".

使用新的可选链接运算符，我们可以重写上面的示例，如下所示：

```js
// Still checks for errors and is much more readable.
const nameLength = db?.user?.name?.length;
```

当`db`,`user`或`name`是`undefined`或`null`?使用可选的链式运算符，JavaScript 初始化`nameLength`自`undefined`而不是抛出错误。

请注意，此行为也比我们的检查更可靠`if (db && db.user && db.user.name)`.例如，如果`name`总是保证是一根绳子？我们可以改变`name?.length`自`name.length`.然后，如果`name`是一个空字符串，我们仍然会得到正确的`0`长度。这是因为空字符串是一个 falsy 值：它的行为类似于`false`在`if`第。可选的链式运算符修复了此常见 Bug 源。

## 其他语法形式：调用和动态属性

还有一个用于调用可选方法的运算符版本：

```js
// Extends the interface with an optional method, which is present
// only for admin users.
const adminOption = db?.user?.validateAdminAndGetPrefs?.().option;
```

语法可能会让人感觉出乎意料，因为`?.()`是实际运算符，适用于表达式*以前*它。

运算符还有第三种用法，即可选的动态属性访问，它是通过以下方式完成的：`?.[]`.它返回括号中的参数所引用的值，或者`undefined`如果没有要从中获取值的对象。下面是一个可能的用例，遵循上面的示例：

```js
// Extends the capabilities of the static property access
// with a dynamically generated property name.
const optionName = 'optional setting';
const optionLength = db?.user?.preferences?.[optionName].length;
```

最后一种形式也可用于可选的索引数组，例如：

```js
// If the `usersArray` is `null` or `undefined`,
// then `userName` gracefully evaluates to `undefined`.
const userIndex = 42;
const userName = usersArray?.[userIndex].name;
```

可选的链式操作器可与[空合并`??`算子](/features/nullish-coalescing)当非`undefined`默认值是必需的。这允许使用指定的默认值进行安全的深层属性访问，从而解决以前需要用户空间库的常见用例，例如[洛达什`_.get`](https://lodash.dev/docs/4.17.15#get):

```js
const object = { id: 123, names: { first: 'Alice', last: 'Smith' }};

{ // With lodash:
  const firstName = _.get(object, 'names.first');
  // → 'Alice'

  const middleName = _.get(object, 'names.middle', '(no middle name)');
  // → '(no middle name)'
}

{ // With optional chaining and nullish coalescing:
  const firstName = object?.names?.first ?? '(no first name)';
  // → 'Alice'

  const middleName = object?.names?.middle ?? '(no middle name)';
  // → '(no middle name)'
}
```

## 可选链接运算符的属性 { #properties }

可选的链式运算符具有一些有趣的属性：*短路*,*堆垛*和*可选删除*.让我们通过一个示例逐一介绍。

*短路*意味着如果可选链接运算符提前返回，则不计算表达式的其余部分：

```js
// `age` is incremented only if `db` and `user` are defined.
db?.user?.grow(++age);
```

*堆垛*意味着可以对一系列属性访问应用多个可选的链式运算符：

```js
// An optional chain may be followed by another optional chain.
const firstNameLength = db.users?.[42]?.names.first.length;
```

不过，在单个链中使用多个可选的链式运算符时，还是要体贴的。如果某个值被保证为空，则使用`?.`不鼓励访问其上的属性。在上面的例子中，`db`被认为总是被定义的，但是`db.users`和`db.users[42]`可能不是。如果数据库中有这样的用户，则`names.first.length`假定始终被定义。

*可选删除*表示`delete`运算符可以与可选链结合使用：

```js
// `db.user` is deleted only if `db` is defined.
delete db?.user;
```

更多细节可以在[这*语义学*提案部分](https://github.com/tc39/proposal-optional-chaining#semantics).

## 支持可选链接 { #support }

<feature-support chrome="80 https://bugs.chromium.org/p/v8/issues/detail?id=9553"
              firefox="74 https://bugzilla.mozilla.org/show_bug.cgi?id=1566143"
              safari="13.1 https://bugs.webkit.org/show_bug.cgi?id=200199"
              nodejs="14 https://medium.com/@nodejs/node-js-version-14-available-now-8170d384567e"
              babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-optional-chaining"></feature-support>
