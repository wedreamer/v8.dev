***

标题： '`Atomics.wait`,`Atomics.notify`,`Atomics.waitAsync`'
作者： '[Marja Hölttä](https://twitter.com/marjakh)，非屏蔽博主
化身：

*   马利亚-霍尔塔
    日期： 2020-09-24
    标签：
*   ECMAScript
*   ES2020
*   节点.js 16
    描述： 'Atomics.wait 和 Atomics.notify 是低级同步原语，可用于实现例如互斥锁。Atomics.wait 仅在工作线程上可用。V8 版本 8.7 现在支持非阻塞版本 Atomics.waitAsync，该版本也可在主线程上使用。
    推文：“1309118447377358848”

***

[`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)和[`Atomics.notify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/notify)是低级同步基元，可用于实现互斥锁和其他同步方式。但是，由于`Atomics.wait`是阻塞的，不可能在主线程上调用它（尝试这样做会抛出一个`TypeError`).

从版本 8.7 开始，V8 支持非阻塞版本，[`Atomics.waitAsync`](https://github.com/tc39/proposal-atomics-wait-async/blob/master/PROPOSAL.md)，这也可以在主线程上使用。

在这篇文章中，我们将解释如何使用这些低级 API 来实现同步（对于工作线程）和异步（对于工作线程或主线程）工作的互斥体。

`Atomics.wait`和`Atomics.waitAsync`采用以下参数：

*   `buffer`：一个`Int32Array`或`BigInt64Array`由`SharedArrayBuffer`
*   `index`：数组中的有效索引
*   `expectedValue`：我们希望存在于描述的内存位置中的值`(buffer, index)`
*   `timeout`：以毫秒为单位的超时（可选，默认为`Infinity`)

的返回值`Atomics.wait`是一个字符串。如果内存位置不包含预期值，`Atomics.wait`立即返回值`'not-equal'`.否则，将阻塞该线程，直到另一个线程调用`Atomics.notify`具有相同的内存位置或达到超时。在前一种情况下，`Atomics.wait`返回值`'ok'`，在后一种情况下，`Atomics.wait`返回值`'timed-out'`.

`Atomics.notify`采用以下参数：

*   一`Int32Array`或`BigInt64Array`由`SharedArrayBuffer`
*   索引（在数组中有效）
*   要通知多少名服务员（可选，默认为`Infinity`)

它以FIFO顺序通知给定数量的服务员，等待由`(buffer, index)`.如果有多个待处理`Atomics.wait`呼叫或`Atomics.waitAsync`与同一位置相关的呼叫，它们都位于同一FIFO队列中。

对比`Atomics.wait`,`Atomics.waitAsync`总是立即返回。返回值为下列值之一：

*   `{ async: false, value: 'not-equal' }`（如果内存位置不包含预期值）
*   `{ async: false, value: 'timed-out' }`（仅适用于即时超时 0）
*   `{ async: true, value: promise }`

稍后可能会使用字符串值解析承诺`'ok'`（如果`Atomics.notify`使用相同的内存位置调用）或`'timed-out'`（如果已达到超时）。承诺永远不会被拒绝。

下面的示例演示了 的基本用法`Atomics.waitAsync`:

```js
const sab = new SharedArrayBuffer(16);
const i32a = new Int32Array(sab);
const result = Atomics.waitAsync(i32a, 0, 0, 1000);
//                                     |  |  ^ timeout (opt)
//                                     |  ^ expected value
//                                     ^ index

if (result.value === 'not-equal') {
  // The value in the SharedArrayBuffer was not the expected one.
} else {
  result.value instanceof Promise; // true
  result.value.then(
    (value) => {
      if (value == 'ok') { /* notified */ }
      else { /* value is 'timed-out' */ }
    });
}

// In this thread, or in another thread:
Atomics.notify(i32a, 0);
```

接下来，我们将展示如何实现可以同步和异步使用的互斥体。前面已经讨论过实现互斥体的同步版本，例如[在这篇博客文章中](https://blogtitle.github.io/using-javascript-sharedarraybuffers-and-atomics/).

在此示例中，我们不在 中使用超时参数`Atomics.wait`和`Atomics.waitAsync`.该参数可用于实现具有超时的条件变量。

我们的互斥体类，`AsyncLock`，在`SharedArrayBuffer`并实现以下方法：

*   `lock`— 阻塞线程，直到我们能够锁定互斥体（仅在工作线程上可用）
*   `unlock`— 解锁互斥体（对应的`lock`)
*   `executeLocked(callback)`— 非阻塞锁，可由主线程使用;附表`callback`一旦我们设法获得锁定，就会执行

让我们看看如何实现其中的每一个。类定义包括常量和一个构造函数，该构造函数采用`SharedArrayBuffer`作为参数。

```js
class AsyncLock {
  static INDEX = 0;
  static UNLOCKED = 0;
  static LOCKED = 1;

  constructor(sab) {
    this.sab = sab;
    this.i32a = new Int32Array(sab);
  }

  lock() {
    /* … */
  }

  unlock() {
    /* … */
  }

  executeLocked(f) {
    /* … */
  }
}
```

这里`i32a[0]`包含值`LOCKED`或`UNLOCKED`.它也是等待位置`Atomics.wait`和`Atomics.waitAsync`.这`AsyncLock`类确保以下不变量：

1.  如果`i32a[0] == LOCKED`，并且线程开始等待（通过`Atomics.wait`或`Atomics.waitAsync`） 上`i32a[0]`，它最终将收到通知。
2.  收到通知后，线程会尝试获取锁。如果它获得锁定，它将在释放它时再次通知。

## 同步锁定和解锁

接下来我们展示阻塞`lock`只能从工作线程调用的方法：

```js
lock() {
  while (true) {
    const oldValue = Atomics.compareExchange(this.i32a, AsyncLock.INDEX,
                        /* old value >>> */  AsyncLock.UNLOCKED,
                        /* new value >>> */  AsyncLock.LOCKED);
    if (oldValue == AsyncLock.UNLOCKED) {
      return;
    }
    Atomics.wait(this.i32a, AsyncLock.INDEX,
                 AsyncLock.LOCKED); // <<< expected value at start
  }
}
```

当线程调用时`lock()`，首先它尝试通过使用`Atomics.compareExchange`将锁定状态从`UNLOCKED`自`LOCKED`.`Atomics.compareExchange`尝试以原子方式执行状态更改，并返回内存位置的原始值。如果原始值为`UNLOCKED`，我们知道状态更改成功，并且线程获取了锁。不需要更多。

如果`Atomics.compareExchange`无法更改锁定状态，必须有另一个线程持有锁定。因此，此线程尝试`Atomics.wait`为了等待另一个线程释放锁。如果内存位置仍包含预期值（在本例中，`AsyncLock.LOCKED`），呼叫`Atomics.wait`将阻塞线程和`Atomics.wait`仅当另一个线程调用时，调用才会返回`Atomics.notify`.

这`unlock`is 方法将锁定设置为`UNLOCKED`状态和调用`Atomics.notify`叫醒一个等待锁的服务员。状态更改始终有望成功，因为此线程持有锁，并且其他人不应调用`unlock()`同时。

```js
unlock() {
  const oldValue = Atomics.compareExchange(this.i32a, AsyncLock.INDEX,
                      /* old value >>> */  AsyncLock.LOCKED,
                      /* new value >>> */  AsyncLock.UNLOCKED);
  if (oldValue != AsyncLock.LOCKED) {
    throw new Error('Tried to unlock while not holding the mutex');
  }
  Atomics.notify(this.i32a, AsyncLock.INDEX, 1);
}
```

简单的情况如下：锁是空闲的，线程T1通过更改锁定状态来获取它`Atomics.compareExchange`.线程 T2 尝试通过调用来获取锁`Atomics.compareExchange`，但它无法成功更改锁定状态。然后呼叫 T2`Atomics.wait`，这会阻塞线程。在某些时候，T1释放锁并呼叫`Atomics.notify`.这使得`Atomics.wait`T2 回车时呼叫`'ok'`，唤醒 T2。然后，T2 尝试再次获取锁，这次成功。

还有2种可能的角落案例 - 这些案例证明了原因`Atomics.wait`和`Atomics.waitAsync`检查索引处的特定值：

*   T1 握住锁，T2 尝试获取锁。首先，T2 尝试更改锁定状态`Atomics.compareExchange`，但没有成功。但随后 T1 在 T2 设法调用之前释放了锁`Atomics.wait`.当 T2 呼叫时`Atomics.wait`，则立即返回值`'not-equal'`.在这种情况下，T2 继续进行下一个循环迭代，尝试再次获取锁。
*   T1 拿着锁，T2 正在等待它`Atomics.wait`.T1 释放锁 — T2 唤醒（`Atomics.wait`调用返回）并尝试执行`Atomics.compareExchange`获取锁，但另一个线程T3更快并且已经获得了锁。所以呼吁`Atomics.compareExchange`未能获得锁定，并且 T2 调用`Atomics.wait`再次阻塞，直到 T3 释放锁。

由于后一种角落情况，互斥体并不“公平”。T2可能一直在等待锁被释放，但T3来了并立即得到它。更现实的锁实现可能使用几种状态来区分“已锁定”和“已争用锁定”。

## 异步锁

非阻塞`executeLocked`方法可以从主线程调用，与阻塞不同`lock`方法。它获取回调函数作为其唯一参数，并计划在成功获取锁后执行回调。

```js
executeLocked(f) {
  const self = this;

  async function tryGetLock() {
    while (true) {
      const oldValue = Atomics.compareExchange(self.i32a, AsyncLock.INDEX,
                          /* old value >>> */  AsyncLock.UNLOCKED,
                          /* new value >>> */  AsyncLock.LOCKED);
      if (oldValue == AsyncLock.UNLOCKED) {
        f();
        self.unlock();
        return;
      }
      const result = Atomics.waitAsync(self.i32a, AsyncLock.INDEX,
                                       AsyncLock.LOCKED);
                                   //  ^ expected value at start
      await result.value;
    }
  }

  tryGetLock();
}
```

内部功能`tryGetLock`尝试首先获得锁`Atomics.compareExchange`如故。如果成功更改锁定状态，它可以执行回调、解锁锁定并返回。

如果`Atomics.compareExchange`未能获得锁，我们需要在锁可能空闲时重试。我们不能阻止并等待锁变得空闲 - 相反，我们使用`Atomics.waitAsync`和它返回的应许。

如果我们成功启动`Atomics.waitAsync`，则返回的承诺在锁定线程执行`Atomics.notify`.然后，等待锁的线程会像以前一样尝试再次获取锁。

相同的角情况（锁在`Atomics.compareExchange`调用和`Atomics.waitAsync`调用，以及在 Promise 解析和`Atomics.compareExchange`call） 在异步版本中也是可能的，因此代码必须以健壮的方式处理它们。

## 结论

在这篇文章中，我们展示了如何使用同步基元`Atomics.wait`,`Atomics.waitAsync`和`Atomics.notify`，以实现可在主线程和工作线程中使用的互斥体。

## 功能支持 { #support }

### `Atomics.wait`和`Atomics.notify`

<feature-support chrome="68"
              firefox="78"
              safari="no"
              nodejs="8.10.0"
              babel="no"></feature-support>

### `Atomics.waitAsync`

<feature-support chrome="87"
              firefox="no"
              safari="no"
              nodejs="16"
              babel="no"></feature-support>
