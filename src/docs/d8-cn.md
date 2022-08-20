***

## 标题： '使用`d8`'&#xA;描述：'d8 是 V8 自己的开发者 shell。

[`d8`](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/d8/)是 V8 自己的开发者 shell。

`d8`对于在本地运行一些 JavaScript 或调试您对 V8 所做的更改非常有用。[使用 GN 构建 V8](/docs/build-gn)对于 x64 输出 a`d8`二进制`out.gn/x64.optdebug/d8`.您可以致电`d8`与`--help`参数，了解有关用法和标志的更多信息。

## 打印到命令行

如果您打算使用，打印输出可能非常重要`d8`以交互方式运行 JavaScript 文件。这可以通过以下方式实现：`console.log`:

```bash
$ cat test.js
console.log('Hello world!');

$ out.gn/x64.optdebug/d8 test.js
Hello world!
```

`d8`还附带一个全球`print`执行相同操作的函数。然而`console.log`优先于`print`因为它也适用于Web浏览器。

## 读取输入

用`read()`您可以将文件的内容存储到变量中。

```js
d8> const license = read('LICENSE');
d8> license
"This license applies to all parts of V8 that are not externally
maintained libraries.  The externally maintained libraries used by V8
are:
… (etc.)"
```

用`readline()`以交互方式输入文本：

```js
d8> const greeting = readline();
Welcome
d8> greeting
"Welcome"
```

## 加载外部脚本

`load()`在当前上下文中运行另一个 JavaScript 文件，这意味着您可以访问该文件中声明的任何内容。

```js
$ cat util.js
function greet(name) {
  return 'Hello, ' + name;
}

$ d8
d8> load('util.js');
d8> greet('World!');
"Hello, World!"
```

## 将标志传递到 JavaScript 中

可以在运行时使命令行参数可用于 JavaScript 代码`d8`.只需在之后通过它们`--`在命令行上。然后，您可以使用`arguments`对象。

```bash
out.gn/x64.optdebug/d8 -- hi
```

现在，您可以使用`arguments`对象：

```js
d8> arguments[0]
"hi"
```

## 更多资源

[凯文·恩尼斯的D8指南](https://gist.github.com/kevincennis/0cd2138c78a07412ef21)有关于探索V8的非常好的信息`d8`.

名称的背景`d8`：很早以前，V8有一个”[样品壳](https://chromium.googlesource.com/v8/v8/+/master/samples/shell.cc)“，其目的是演示如何嵌入 V8 来构建 JavaScript shell。它是故意极简主义的，简单地称为“shell”。不久之后，增加了一个“开发者 shell”，它具有更多便利功能，以帮助开发人员进行日常工作，它也需要一个名字。选择“d8”作为名称的原始原因已消失在历史中;它流行起来是因为“eveloper”是8个省略的字符，所以“d8 shell”作为缩写是有意义的，并且也非常适合“V8”作为项目名称。
