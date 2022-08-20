***

标题： '`Intl.ListFormat`'
作者： 马蒂亚斯·拜恩斯 （[@mathias](https://twitter.com/mathias)）和弗兰克·唐
化身：

*   'mathias-bynens'
*   “坦唐”
    日期： 2018-12-18
    标签：
*   国际
*   节点.js 12
*   io19
    描述： 'Intl.ListFormat API 支持在不牺牲性能的情况下对列表进行本地化格式化。
    推文：“1074966915557351424”

***

现代 Web 应用程序通常使用由动态数据组成的列表。例如，照片查看器应用可能会显示如下内容：

> 这张照片包括**艾达， 伊迪丝，*和*恩典**.

基于文本的游戏可能具有不同类型的列表：

> 选择您的超能力：**隐形、精神运动、*或*移情**.

由于每种语言都有不同的列表格式约定和单词，因此实现本地化的列表格式化程序并非易事。这不仅需要列出您要支持的每种语言的所有单词（例如上面示例中的“and”或“or”），此外，您还需要对所有这些语言的确切格式约定进行编码！[The Unicode CLDR](http://cldr.unicode.org/translation/lists)提供了这些数据，但是要在JavaScript中使用它，它必须嵌入并与其他库代码一起发布。不幸的是，这会增加此类库的捆绑包大小，从而对加载时间、解析/编译成本和内存消耗产生负面影响。

全新`Intl.ListFormat`API将这种负担转移到JavaScript引擎上，JavaScript引擎可以发布区域设置数据，并使其直接提供给JavaScript开发人员。`Intl.ListFormat`启用列表的本地化格式设置，而不会牺牲性能。

## 使用示例

下面的示例演示如何使用英语为连词创建列表格式化程序：

```js
const lf = new Intl.ListFormat('en');
lf.format(['Frank']);
// → 'Frank'
lf.format(['Frank', 'Christine']);
// → 'Frank and Christine'
lf.format(['Frank', 'Christine', 'Flora']);
// → 'Frank, Christine, and Flora'
lf.format(['Frank', 'Christine', 'Flora', 'Harrison']);
// → 'Frank, Christine, Flora, and Harrison'
```

析取（英语中的“or”）也通过可选的`options`参数：

```js
const lf = new Intl.ListFormat('en', { type: 'disjunction' });
lf.format(['Frank']);
// → 'Frank'
lf.format(['Frank', 'Christine']);
// → 'Frank or Christine'
lf.format(['Frank', 'Christine', 'Flora']);
// → 'Frank, Christine, or Flora'
lf.format(['Frank', 'Christine', 'Flora', 'Harrison']);
// → 'Frank, Christine, Flora, or Harrison'
```

下面是使用其他语言（中文，带有语言代码）的示例`zh`):

```js
const lf = new Intl.ListFormat('zh');
lf.format(['永鋒']);
// → '永鋒'
lf.format(['永鋒', '新宇']);
// → '永鋒和新宇'
lf.format(['永鋒', '新宇', '芳遠']);
// → '永鋒、新宇和芳遠'
lf.format(['永鋒', '新宇', '芳遠', '澤遠']);
// → '永鋒、新宇、芳遠和澤遠'
```

这`options`参数启用更高级的用法。下面概述了各种选项及其组合，以及它们如何与定义的列表模式相对应[美国医学科学院院士#35](https://unicode.org/reports/tr35/tr35-general.html#ListPatterns):

：：：表包装器
|类型 |选项|描述 |示例 |
|--------------------- |----------------------------------------- |----------------------------------------------------------------------------------------------- |-------------------------------- |
|标准（或无类型）|`{}`（默认值）|任意占位符的典型“and”列表|`'January, February, and March'`|
|或|`{ type: 'disjunction' }`|任意占位符的典型“or”列表|`'January, February, or March'`|
|单位 |`{ type: 'unit' }`|适用于宽单位|的列表`'3 feet, 7 inches'`|
|单位短|`{ type: 'unit', style: 'short' }`|适用于短单位|的列表`'3 ft, 7 in'`|
|单元窄|`{ type: 'unit', style: 'narrow' }`|适用于窄单元的列表，其中屏幕上的空间非常有限|`'3′ 7″'`|
:::

请注意，在许多语言（如英语）中，许多这些列表之间可能没有区别。在其他情况下，连词的间距、长度或存在以及分隔符可能会发生变化。

## 结论

作为`Intl.ListFormat`API 变得越来越普遍，您会发现库放弃了对硬编码 CLDR 数据库的依赖性，转而使用本机列表格式设置功能，从而提高了加载时性能、解析和编译时性能、运行时性能和内存使用率。

## `Intl.ListFormat`支持 { #support }

<feature-support chrome="72 /blog/v8-release-72#intl.listformat"
              firefox="no"
              safari="no"
              nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
              babel="no"></feature-support>
