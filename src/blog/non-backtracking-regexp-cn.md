***

标题： “一个额外的非回溯正则表达式引擎”
作者： 'Martin Bidlingmaier'
日期： 2021-01-11
标签：

*   内部
*   正则表达式
    描述： “V8 现在有一个额外的 RegExp 引擎，可以作为回退，防止许多灾难性回溯的实例。
    推文：“1348635270762139650”

***

从 v8.8 开始，V8 配备了一个新的实验性非回溯 RegExp 引擎（除了现有的[Irregexp engine](https://blog.chromium.org/2009/02/irregexp-google-chromes-new-regexp.html)）， 它保证相对于主题字符串大小的线性时间执行。实验引擎位于下面提到的功能标志后面。

![Runtime of /(a\*)\*b/.exec('a'.repeat(n)) for n ≤ 100](/\_img/non-backtracking-regexp/runtime-plot.svg)

下面介绍如何配置新的正则表达式引擎：

*   `--enable-experimental-regexp_engine-on-excessive-backtracks`允许在过多的回溯时回退到非回溯引擎。
*   `--regexp-backtracks-before-fallback N`（默认值 N = 50，000）指定有多少回溯被视为“过多”，即回退何时启动。
*   `--enable-experimental-regexp-engine`开启非标识别`l`（“线性”） 正则表达式的标志，例如`/(a*)*b/l`.使用此标志构造的正则表达式总是使用新引擎热切地执行;Irregexp根本不参与其中。如果新的 RegExp 引擎无法处理`l`-RegExp，则在构造时引发异常。我们希望此功能在某些时候可用于强化在不受信任的输入上运行正则表达式的应用。目前，它仍然是实验性的，因为在最常见的模式上，Irregexp比新引擎快几个数量级。

回退机制不适用于所有模式。要启动回退机制，正则表达式必须：

*   不包含反向引用，
*   不包含前瞻或前瞻，
*   不包含大型或深度嵌套的有限重复，例如`/a{200,500}/`和
*   没有`u`（Unicode） 或`i`（不区分大小写）标志集。

## 背景：灾难性的回溯

V8 中的 RegExp 匹配由 Irregexp 引擎处理。Irregexp jit-compile RegExps 为专门的本机代码（或[字节码](/blog/regexp-tier-up)），因此对于大多数模式来说非常快。但是，对于某些模式，Irregexp 的运行时可能会在输入字符串的大小上呈指数级增长。上面的例子，`/(a*)*b/.exec('a'.repeat(100))`，如果由 Irregexp 执行，则不会在我们的有生之年完成。

这到底是怎么回事呢？Irregexp 是一个*回溯*发动机。当面临如何继续比赛的选择时，Irregexp会全面探索第一种选择，然后在必要时回溯以探索第二种选择。例如，考虑匹配模式`/abc|[az][by][0-9]/`针对主题字符串`'ab3'`.在这里，Irregexp试图匹配`/abc/`第一个字符，在第二个字符之后失败。然后，它回溯两个字符并成功匹配第二个备选项`/[az][by][0-9]/`.在带有量词的模式中，例如`/(abc)*xyz/`，Irregexp 必须在匹配身体后选择是再次匹配身体还是继续剩余的模式。

让我们尝试了解匹配时发生了什么`/(a*)*b/`针对较小的主题字符串，例如`'aaa'`.此模式包含嵌套量词，因此我们要求 Irregexp 匹配*序列序列*之`'a'`，然后匹配`'b'`.显然没有匹配项，因为主题字符串不包含`'b'`.然而`/(a*)*/`匹配，它以指数级多种不同的方式做到这一点：

```js
'aaa'           'aa', 'a'           'aa', ''
'a', 'aa'       'a', 'a', 'a'       'a', 'a', ''
…
```

先验地，Irregexp不能排除未能匹配决赛的可能性。`/b/`是由于选择了错误的匹配方式`/(a*)*/`，所以它必须尝试所有变体。这个问题被称为“指数”或“灾难性”回溯。

## 作为自动机和字节码的正则表达式

要了解一种不受灾难性回溯影响的替代算法，我们必须通过以下方式快速绕道而行[胞 自动 机](https://en.wikipedia.org/wiki/Nondeterministic_finite_automaton).每个正则表达式都等效于一个自动机。例如，正则表达式`/(a*)*b/`以上对应于以下自动机：

![Automaton corresponding to /(a\*)\*b/](/\_img/non-backtracking-regexp/example-automaton.svg)

请注意，自动机不是由模式唯一确定的;您在上面看到的是您将通过机械翻译过程获得的自动机，它是V8的新RegExp引擎中使用的自动机。`/(a*)*/`.
未标记的边缘是 epsilon 转换：它们不消耗输入。Epsilon 转换对于将自动机的大小保持在阵列大小附近是必要的。天真地消除ε跃迁会导致跃迁次数的二次增加。
Epsilon 转换还允许从以下四种基本状态构造对应于 RegExp 的自动机：

![RegExp bytecode instructions](/\_img/non-backtracking-regexp/state-types.svg)

这里我们只对过渡进行分类*外*的状态，而向状态的转换仍然被允许是任意的。仅从这些类型的状态构建的自动机可以表示为*字节码程序*，其中每个状态都对应于一条指令。例如，具有两个 epsilon 转换的状态表示为`FORK`指令。

## 回溯算法

让我们重新审视Irregexp所基于的回溯算法，并用自动机来描述它。假设我们得到一个字节码数组`code`对应于模式和想要`test`是否`input`与模式匹配。让我们假设`code`看起来像这样：

```js
const code = [
  {opcode: 'FORK', forkPc: 4},
  {opcode: 'CONSUME', char: '1'},
  {opcode: 'CONSUME', char: '2'},
  {opcode: 'JMP', jmpPc: 6},
  {opcode: 'CONSUME', char: 'a'},
  {opcode: 'CONSUME', char: 'b'},
  {opcode: 'ACCEPT'}
];
```

此字节码对应于（粘性）模式`/12|ab/y`.这`forkPc`的领域`FORK`指令是我们可以继续的替代状态/指令的索引（“程序计数器”），类似地`jmpPc`.索引从零开始。回溯算法现在可以在JavaScript中实现，如下所示。

```js
let ip = 0; // Input position.
let pc = 0; // Program counter: index of the next instruction.
const stack = []; // Backtrack stack.
while (true) {
  const inst = code[pc];
  switch (inst.opcode) {
    case 'CONSUME':
      if (ip < input.length && input[ip] === inst.char) {
        // Input matches what we expect: Continue.
        ++ip;
        ++pc;
      } else if (stack.length > 0) {
        // Wrong input character, but we can backtrack.
        const back = stack.pop();
        ip = back.ip;
        pc = back.pc;
      } else {
        // Wrong character, cannot backtrack.
        return false;
      }
      break;
    case 'FORK':
      // Save alternative for backtracking later.
      stack.push({ip: ip, pc: inst.forkPc});
      ++pc;
      break;
    case 'JMP':
      pc = inst.jmpPc;
      break;
    case 'ACCEPT':
      return true;
  }
}
```

如果字节码程序包含不消耗任何字符的循环，即如果自动机包含仅由 epsilon 转换组成的循环，则此实现将无限期循环。这个问题可以通过单个字符来解决。Irregexp比这个简单的实现复杂得多，但最终基于相同的算法。

## 非回溯算法

回溯算法对应于*深度优先*自动机的遍历：我们总是探索自动机的第一个替代方案`FORK`语句完整，然后在必要时回溯到第二个备选项。因此，非回溯算法的替代方案并不奇怪地基于*广度优先*自动机的遍历。在这里，我们同时考虑所有备选项，相对于输入字符串中的当前位置同步。因此，我们维护一个当前状态列表，然后通过获取对应于每个输入字符的转换来推进所有状态。至关重要的是，我们从当前状态列表中删除重复项。

JavaScript 中的一个简单的实现如下所示：

```js
// Input position.
let ip = 0;
// List of current pc values, or `'ACCEPT'` if we’ve found a match. We start at
// pc 0 and follow epsilon transitions.
let pcs = followEpsilons([0]);

while (true) {
  // We’re done if we’ve found a match…
  if (pcs === 'ACCEPT') return true;
  // …or if we’ve exhausted the input string.
  if (ip >= input.length) return false;

  // Continue only with the pcs that CONSUME the correct character.
  pcs = pcs.filter(pc => code[pc].char === input[ip]);
  // Advance the remaining pcs to the next instruction.
  pcs = pcs.map(pc => pc + 1);
  // Follow epsilon transitions.
  pcs = followEpsilons(pcs);

  ++ip;
}
```

这里`followEpsilons`是一个函数，它获取程序计数器列表并计算程序计数器列表`CONSUME`可以通过 epsilon 转换（即仅执行 FORK 和 JMP）到达的指令。返回的列表不得包含重复项。如果`ACCEPT`指令可以到达，函数返回`'ACCEPT'`.它可以像这样实现：

```js
function followEpsilons(pcs) {
  // Set of pcs we’ve seen so far.
  const visitedPcs = new Set();
  const result = [];

  while (pcs.length > 0) {
    const pc = pcs.pop();

    // We can ignore pc if we’ve seen it earlier.
    if (visitedPcs.has(pc)) continue;
    visitedPcs.add(pc);

    const inst = code[pc];
    switch (inst.opcode) {
      case 'CONSUME':
        result.push(pc);
        break;
      case 'FORK':
        pcs.push(pc + 1, inst.forkPc);
        break;
      case 'JMP':
        pcs.push(inst.jmpPc);
        break;
      case 'ACCEPT':
        return 'ACCEPT';
    }
  }

  return result;
}
```

由于通过`visitedPcs`设置，我们知道每个程序计数器只在`followEpsilons`.这保证了`result`list 不包含重复项，并且`followEpsilons`受`code`数组，即模式的大小。`followEpsilons`最多被调用`input.length`次，因此 RegExp 匹配的总运行时受以下限制：`𝒪(pattern.length * input.length)`.

非回溯算法可以扩展以支持JavaScript RegExps的大多数功能，例如单词边界或（子）匹配边界的计算。不幸的是，如果不进行重大更改，改变渐近最坏情况的复杂性，就无法支持反向引用、前瞻和后视。

V8 的新 RegExp 引擎基于此算法及其在[re2](https://github.com/google/re2)和[Rust regex](https://github.com/rust-lang/regex)图书馆。该算法的讨论比这里深入得多，非常出色[系列博客文章](https://swtch.com/~rsc/regexp/)作者：Russ Cox，他也是 re2 库的原始作者。
