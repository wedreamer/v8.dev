***

## 标题： “GUI 和 IDE 设置”&#xA;描述：“本文档包含有关处理 V8 代码库的 GUI 和特定于 IDE 的提示。

V8 源代码可以在线浏览[铬代码搜索](https://cs.chromium.org/chromium/src/v8/).

可以使用许多其他客户端程序和插件访问此项目的 Git 存储库。有关详细信息，请参阅客户的文档。

## Visual Studio Code 和 clangd

有关如何为 V8 设置 VSCode 的说明，请参阅以下内容[公文](https://docs.google.com/document/d/1BpdCFecUGuJU5wN6xFkHQJEykyVSlGN8B9o3Kz2Oes8/).这是当前（2021）推荐的设置。

## 日蚀

有关如何为 V8 设置 Eclipse 的说明，请参阅以下内容[公文](https://docs.google.com/document/d/1q3JkYNJhib3ni9QvNKIY_uarVxeVDiDi6teE5MbVIGQ/).注意：截至 2020 年，使用 Eclipse 索引 V8 不能很好地工作。

## Visual Studio Code and cquery

VSCode 和 cquery 提供了良好的代码导航功能。它为C++符号提供了“转到定义”以及“查找所有参考”，并且效果很好。本节介绍如何在 \*nix 系统上进行基本设置。

### 安装 VSCode

以首选方式安装 VSCode。本指南的其余部分假定你可以通过以下命令从命令行运行 VSCode`code`.

### 安装查询

克隆查询从[查询](https://github.com/cquery-project/cquery)在您选择的目录中。我们使用`CQUERY_DIR="$HOME/cquery"`在本指南中。

```bash
git clone https://github.com/cquery-project/cquery "$CQUERY_DIR"
cd "$CQUERY_DIR"
git submodule update --init
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=release -DCMAKE_INSTALL_PREFIX=release -DCMAKE_EXPORT_COMPILE_COMMANDS=YES
make install -j8
```

如果出现任何问题，请务必退房[查询的入门指南](https://github.com/cquery-project/cquery/wiki).

您可以使用`git pull && git submodule update`在以后更新 cquery（不要忘记通过以下方式重建`cmake .. -DCMAKE_BUILD_TYPE=release -DCMAKE_INSTALL_PREFIX=release -DCMAKE_EXPORT_COMPILE_COMMANDS=YES && make install -j8`).

### 为 VSCode 安装和配置 cquery 插件

在 VSCode 中从市场安装 cquery 扩展。在 V8 签出中打开 VSCode：

```bash
cd v8
code .
```

转到 VSCode 中的设置，例如，通过快捷方式<kbd>Ctrl</kbd>+<kbd>,</kbd>.

将以下内容添加到工作区配置中，以替换`YOURUSERNAME`和`YOURV8CHECKOUTDIR`适当地。

```json
"settings": {
  "cquery.launch.command": "/home/YOURUSERNAME/cquery/build/release/bin/cquery",
  "cquery.cacheDirectory": "/home/YOURUSERNAME/YOURV8CHECKOUTDIR/.vscode/cquery_cached_index/",
  "cquery.completion.include.blacklist": [".*/.vscache/.*", "/tmp.*", "build/.*"],
  […]
}
```

### 提供`compile_commands.json`进行查询

最后一步是生成一个compile_commands.json进行查询。此文件将包含用于将 V8 构建为 cquery 的特定编译器命令行。在 V8 检出中运行以下命令：

```bash
ninja -C out.gn/x64.release -t compdb cxx cc > compile_commands.json
```

这需要不时地重新执行，以教授有关新源文件的查询。特别是，应始终在`BUILD.gn`已更改。

### 其他有用的设置

Visual Studio Code 中括号的自动关闭效果不佳。它可以禁用

```json
"editor.autoClosingBrackets": false
```

在用户设置中。

以下排除掩码有助于避免在使用搜索 （<kbd>Ctrl</kbd>+<kbd>转变</kbd>+<kbd>F</kbd>):

```js
"files.exclude": {
  "**/.vscode": true,  // this is a default value
},
"search.exclude": {
  "**/out*": true,     // this is a default value
  "**/build*": true    // this is a default value
},
```
