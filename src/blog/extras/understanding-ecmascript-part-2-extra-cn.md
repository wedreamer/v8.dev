***

标题： “了解 ECMAScript 规范，第 2 部分”的额外内容”
作者： '[Marja Hölttä](https://twitter.com/marjakh)，投机规格观众
化身：

*   马利亚-霍尔塔
    日期： 2020-03-02
    标签：
*   ECMAScript
    描述： '关于阅读 ECMAScript 规范的教程'
    推特： ''

***

### 为什么是`o2.foo`一`AssignmentExpression`?

`o2.foo`看起来不像`AssignmentExpression`因为没有任务。为什么它是一个`AssignmentExpression`?

该规范实际上允许`AssignmentExpression`既作为参数，也作为作业的右侧。例如：

```js
function simple(a) {
  console.log('The argument was ' + a);
}
simple(x = 1);
// → Logs “The argument was 1”.
x;
// → 1
```

...和。。。

```js
x = y = 5;
x; // 5
y; // 5
```

`o2.foo`是一个`AssignmentExpression`这不会分配任何东西。这遵循以下语法作品，每个语法作品都采用“最简单”的情况，直到最后一个：

一`AssignmentExpresssion`不需要有作业，也可以只是一个`ConditionalExpression`:

> **[`AssignmentExpression : ConditionalExpression`](https://tc39.es/ecma262/#sec-assignment-operators)**

（还有其他作品，这里我们只显示相关的作品。

一个`ConditionalExpression`不需要有条件 （`a == b ? c : d`），它也可以是一个`ShortcircuitExpression`:

> **[`ConditionalExpression : ShortCircuitExpression`](https://tc39.es/ecma262/#sec-conditional-operator)**

等等：

> [`ShortCircuitExpression : LogicalORExpression`](https://tc39.es/ecma262/#prod-ShortCircuitExpression)
>
> [`LogicalORExpression : LogicalANDExpression`](https://tc39.es/ecma262/#prod-LogicalORExpression)
>
> [`LogicalANDExpression : BitwiseORExpression`](https://tc39.es/ecma262/#prod-LogicalANDExpression)
>
> [`BitwiseORExpression : BitwiseXORExpression`](https://tc39.es/ecma262/#prod-BitwiseORExpression)
>
> [`BitwiseXORExpression : BitwiseANDExpression`](https://tc39.es/ecma262/#prod-BitwiseXORExpression)
>
> [`BitwiseANDExpression : EqualityExpression`](https://tc39.es/ecma262/#prod-BitwiseANDExpression)
>
> [`EqualityExpression : RelationalExpression`](https://tc39.es/ecma262/#sec-equality-operators)
>
> [`RelationalExpression : ShiftExpression`](https://tc39.es/ecma262/#prod-RelationalExpression)

快到了。。。

> [`ShiftExpression : AdditiveExpression`](https://tc39.es/ecma262/#prod-ShiftExpression)
>
> [`AdditiveExpression : MultiplicativeExpression`](https://tc39.es/ecma262/#prod-AdditiveExpression)
>
> [`MultiplicativeExpression : ExponentialExpression`](https://tc39.es/ecma262/#prod-MultiplicativeExpression)
>
> [`ExponentialExpression : UnaryExpression`](https://tc39.es/ecma262/#prod-ExponentiationExpression)

不要绝望！只是再制作几部...

> [`UnaryExpression : UpdateExpression`](https://tc39.es/ecma262/#prod-UnaryExpression)
>
> [`UpdateExpression : LeftHandSideExpression`](https://tc39.es/ecma262/#prod-UpdateExpression)

然后我们点击制作`LeftHandSideExpression`:

> [`LeftHandSideExpression :`](https://tc39.es/ecma262/#prod-LeftHandSideExpression)
> `NewExpression`
> `CallExpression`
> `OptionalExpression`

目前尚不清楚哪种生产可能适用于`o2.foo`.我们只需要知道（或找出）一个`NewExpression`实际上不必具有`new`关键词。

> [`NewExpression : MemberExpression`](https://tc39.es/ecma262/#prod-NewExpression)

`MemberExpression`听起来像是我们一直在寻找的东西，所以现在我们开始制作

> [`MemberExpression : MemberExpression . IdentifierName`](https://tc39.es/ecma262/#prod-MemberExpression)

所以`o2.foo`是一个`MemberExpression`如果`o2`是有效的`MemberExpression`.幸运的是，它更容易看到：

> [`MemberExpression : PrimaryExpression`](https://tc39.es/ecma262/#prod-MemberExpression)
>
> [`PrimaryExpression : IdentifierReference`](https://tc39.es/ecma262/#prod-PrimaryExpression)
>
> [`IdentifierReference : Identifier`](https://tc39.es/ecma262/#prod-IdentifierReference)

`o2`肯定是`Identifier`所以我们很好。`o2`是一个`MemberExpression`所以`o2.foo`也是一个`MemberExpression`.一个`MemberExpression`是有效的`AssignmentExpression`所以`o2.foo`是一个`AssignmentExpression`太。
