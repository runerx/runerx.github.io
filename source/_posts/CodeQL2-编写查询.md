---
title: CodeQL2-编写查询
tags:
  - CodeQL
  - 代码安全
categories: 代码安全
date: 2022-02-22 15:17:31
---


<font style="color:Gray; float:left">南有乔木，不可休思；汉有游女，不可求思。</font><br>

<font style="color:Gray; float:right">——《诗经.汉广》</font>

<br>

编写CodeQL查询语句，参考官网文档进行翻译。

<!-- more -->





## 1. 关于CodeQL查询

CodeQL查询用于分析与安全性、正确性、可维护性和可读性相关的代码问题。

### 1.1 概述

CodeQL自带的查询语句已经包含了用于查询支持的语言的最重要最有趣的问题。但是，你还是可以自己编写查询语句用来查询自己项目中你关心的问题。CodeQL的查询语句类型如下：

- `Alert queries`: 查询你代码中明显的代码问题。
- `Path queries`:  用于查询source和sink点之前的数据流信息。

可以将自己编写的CodeQL查询添加到`QL packs`，然后使用[Code scanning](https://docs.github.com/en/code-security/secure-coding/automatically-scanning-your-code-for-vulnerabilities-and-errors/about-code-scanning)扫描，或者CodeQL CLI扫描或者添加到CodeQL查询的开源github仓库。

这一节主要是基础知识，更多CodeQL查询相关的内容需要与编程语言结合，官方文档在[CodeQL language guides](https://codeql.github.com/docs/codeql-language-guides/#codeql-language-guides)。关于QL语言的更详细的介绍在[QL language reference](https://codeql.github.com/docs/ql-language-reference/#ql-language-reference)。如果想将自己编写的查询语句贡献给CodeQL查询github仓库，需要遵循一定的规范，相关内容在 [CodeQL style guide](https://github.com/github/codeql/blob/main/docs/ql-style-guide.md)。

### 1.2 基本查询结构

CodeQL查询文件的扩展名是.ql，查询代码需要包含select语句。

现有的查询代码都有如下结构

```
/**
 *
 * Query metadata 
 * 查询元数据
 *
 */

import /* ... CodeQL libraries or modules CodeQL的库或者模块... */

/* ... Optional, define CodeQL classes and predicates 可选内容, 定义CodeQL类和谓词 ... */

from /* ... variable declarations 变量声明 ... */
where /* ... logical formula 逻辑公司... */
select /* ... expressions 表达式... */
```

接下来的几个部分以`Alert queries`为例，讲一讲上述结构的具体内容。`Path queries`的内容会在后门介绍。

**Query metadata**

Query metadata（查询元数据）。

当自己编写的查询文件添加到github仓库或者用于自己分析的时候，需要`Query metadata`来确认相关信息。`Query metadata`包含查询的目的，以及包含如何解释和显示查询结果。

元数据内容取决于你怎样运行你的查询。

- 如果是贡献给GitHub仓库，阅读 [query metadata style guide](https://github.com/github/codeql/blob/main/docs/query-metadata-style-guide.md)
- 如果是添加到LGTM 的查询包里面，阅读[Writing custom queries to include in LGTM analysis](https://lgtm.com/help/lgtm/writing-custom-queries)
- 如果使用CodeQL CLI，元数据必须包含 `@kind`字段
- 如果在LGTM的console或者VS Code扩展运行，metadata是不强制编写的。然而，如果想你的查询结果是`Alert queries`或者`Path queries`类型，必须要有 `@kind`字段。更多信息，LGTM.com参考 [Using the query console](https://lgtm.com/help/lgtm/using-query-console) ，使用VS Code扩展参考[Analyzing your projects](https://codeql.github.com/docs/codeql-for-visual-studio-code/analyzing-your-projects/#analyzing-your-projects)。

> 不管是将查询代码添加到开源仓库，或者添加到LGTM的pack里面，还是使用CodeQL CLi都必须有@kind字段，@kind指明怎样解释和显示查询结果
>
> - Alert query metadat 必须包含 `@kind problem`来确保结果是 simple alert
> - Path query metadata 必须包含 `@kind path-problem` 来确保结果是代码传播过程
> - Diagnostic query metadata 必须包含  `@kind diagnostic` 来确保结果是故障排除
> - Summary query metadata 必须包含  `@kind metric`和 `@tags summary` 来确保结果是CodeQL数据库的概览指标

**Import 语句**

每一个查询代码通常会包含一个或者更多`import`语句，用来定义导入什么库或者模块。库和模块将一组相关的`type`、`predicates`已经其他模块组织在一起。然后查询就可以访问每一个导入的库和模块。 [open source repository on GitHub](https://github.com/github/codeql)包含了支持的每一种语言的标准库。

当编写`alert queries`的时候，通常会导入项目使用的编程语言的标准库。`import`后跟如下语言

- C/C++: `cpp`
- C#: `csharp`
- Go: `go`
- Java: `java`
- JavaScript/TypeScript: `javascript`
- Python: `python`

还有一些库包含常用的`predicates`,`types`,和其他与分析相关的模块，包括`data flow`, `control flow`,和 `taint-tracking`



**可选的CodeQL class和predicates**

可以在查询中编写自己的class和predicates，更多详细类容 [Defining a predicate](https://codeql.github.com/docs/ql-language-reference/predicates/#defining-a-predicate) 和 [Defining a class](https://codeql.github.com/docs/ql-language-reference/types/#defining-a-class)



**From子句**

`from`子句声明查询中要使用的变量，每个声明按照格式：<type> <variable name>，更多内容参考QL语言部分。



**Where子句**

`where`子句定义逻辑条件。

**Select子句**

`select`子句用于显示复合`where`子句条件的`from`子句声明的变量。显示的结构由@kind决定。

如果是`@kind problem`,由两个字段组成：

```
select element, string
```

- `element`: 由查询确定，它就是要查询的内容
- `string`: 一条消息，也可以包含链接和占位符，起解释作用

其他@kind类型在后面将讲到。







## 2. Metadata

Metadata(元数据)告诉用户的重要信息。你必须编写正确的查询元数据来获取结果。

### 2.1 关于查询元数据

Metadata作为QLDodc注释出现在查询文件的顶部。metadata用于定义查询方式、显示方式，以及方便其他用户了解此查询文件的解释。

### 2.2 Metadata属性

下面的属性支持所有的查询文件

| Property             | Value                                                        | Description                                                  |
| :------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `@description`       | `<text>`                                                     | 一条语句或一个段落，用来描述查询的目的和结果的有效性和重要性。用`''`包含 |
| `@id`                | `<text>`                                                     | 由小写字母或数字组成的一系列单词，用`/`或`-`分割，识别和分类查询，每一个查询必须有唯一的ID。为了确保唯一ID，使用固定结构也许有帮助，比如LGTM的标准查询的格式： `<language>/<brief-description>` |
| `@kind`              | `problem`<br>`path-problem`                                  | 确实查询是alert (`@kind problem`) 或者 path (`@kind path-problem`)。 |
| `@name`              | `<text>`                                                     | 定义查询标签的语句。用`''`包含                               |
| `@tags`              | `correctness`<br>`maintainability`<br>`readability`<br>`security` | 这些标签将查询按大的类别分组，以便于搜索和识别它们。除了列出的常见标签，还有一些更具体的类别。参考[Query metadata style guide](https://github.com/github/codeql/blob/main/docs/query-metadata-style-guide.md). |
| `@precision`         | `low`<br>`medium`<br>`high`<br>`very-high`                   | 指示查询结果为正的百分比。这一点，和@problem.severity特性，决定是否在LGTM上默认显示 |
| `@problem.severity`  | `error`<br>`warning`<br>`recommendation`                     | 定义非安全查询的告警程度。和 `@precision`一起，决定是否显示结果 |
| `@security-severity` | `<score>`                                                    | 定义紧急程度，在0.0和10.0之间                                |



### 2.3 filter查询的其他属性

过滤器查询用于定义其他约束，以限制其他查询返回的结果。一个过滤器查询必须有一样的@kind属性，用来指定结果的过滤。不需要其他元数据属性。



### 2.4 用例

标准的java查询元数据

![](https://codeql.github.com/docs/_images/query-metadata.png)



## 3. 查询帮助文件

Query help files（查询帮助文件）告诉用户查询的目的，并建议如何解决查询发现的潜在问题。

这个部分关于查询帮助文件结构的详细信息。关于如何写一个好的查询帮助文件参考 [Query help style guide](https://github.com/github/codeql/blob/main/docs/query-help-style-guide.md)

### 3.1 概览

每个查询帮助文件提供了关于使用一个查询的目的和使用的详细信息。当你编写你自己的查询的时候，我们也建议你也编写一个查询帮助文件以让其他使用者知道这个查询是做什么的已经怎么做的。

### 3.2 结构

查询帮助文档使用自定义的XML格式，存储文件是.qhelp扩展名。查询帮助文件的名字必须根查询文件名称一样，并且在同一目录。基本结果如下：

```
<!DOCTYPE qhelp SYSTEM "qhelp.dtd">
<qhelp>
    CONTAINS one or more section-level elements
</qhelp>
```

头和顶级元素是必须要有的，

### 3.3 Section-level elements

Section-level 元素被用于在帮助文件中将信息组织成不同的部分。许多sections有heading，被 `title`属性或者默认值定义。下面的的section-level 元素可以在child 元素中使用。

| Element          | Attributes                            | Children          | Purpose of section                                           |
| :--------------- | :------------------------------------ | :---------------- | :----------------------------------------------------------- |
| `example`        | None                                  | Any block element | 编写一个用例，并且给出修复方案。默认                         |
| `fragment`       | None                                  | Any block element | 参考 “[Query help inclusion](https://codeql.github.com/docs/writing-codeql-queries/query-help-files/#qhelp-inclusion)” below. |
| `hr`             | None                                  | None              | A horizontal rule. No heading.                               |
| `include`        | `src` The query help file to include. | None              | Include a query help file at the location of this element. See “[Query help inclusion](https://codeql.github.com/docs/writing-codeql-queries/query-help-files/#qhelp-inclusion)” below. |
| `overview`       | None                                  | Any block element | 查询目的概述。通常这是查询文档的第一部分。                   |
| `recommendation` | None                                  | Any block element | 建议如何处理此查询识别的告警。默认标签.                      |
| `references`     | None                                  | `li` elements     | 参考列表，一般在文档最后。默认标签                           |
| `section`        | `title` Title of the section          | Any block element | General-purpose section with a heading defined by the `title` attribute。 |
| `semmleNotes`    | None                                  | Any block element | Implementation notes about the query. This section is used only for queries that implement a rule defined by a third party. Default heading.关于查询的实现说明 |



### 3.4 块元素

下面的元素是 `section`, `example`, `fragment`, `recommendation`, `overview`, and `semmleNotes` 的子元素

| Element      | Attributes                                                   | Children           | Purpose of block                                             |
| :----------- | :----------------------------------------------------------- | :----------------- | :----------------------------------------------------------- |
| `blockquote` | None                                                         | Any block element  | 显示引用的段落                                               |
| `img`        | `src` The image file to include.`alt` Text for the image’s alt text.`height` Optional, height of the image.`width` Optional, the width of the image. | None               | 显示图像。图像的内容位于单独的图像文件中。                   |
| `include`    | `src` The query help file to include.                        | None               | Include a query help file at the location of this element. See [Query help inclusion](https://codeql.github.com/docs/writing-codeql-queries/query-help-files/#qhelp-inclusion) below for more information. |
| `ol`         | None                                                         | `li`               | 显示有序列表。请参见下面的列表元素。                         |
| `p`          | None                                                         | Any inline content | 显示段落，如在HTML文件中使用。                               |
| `pre`        | None                                                         | Text               | 等距字体显示文本                                             |
| `sample`     | `language` The language of the in-line code sample.`src` Optional, the file containing the sample code. | Text               | 当指定了src文件，则根据文件扩展名推断语言，如果没有src文件，必须用sample标签声明 |
| `table`      | None                                                         | `tbody`            | 显示列表                                                     |
| `ul`         | None                                                         | `li`               | 显示无序列表                                                 |
| `warning`    | None                                                         | Text               | 显示明显的告警，用于在较低准确度的查询，这样的查询一般默认禁用 |



### 3.5 List 元素

查询帮助文件支持两种列表元素：: `ul` 和 `ol`。两个块元素都只支持`li`的子元素。每个`li`元素包含内联元素和块元素。



### 3.6 表元素

 `table` 用于在查询帮助文件中生成一个表。

| Element | Attributes | Children           | Purpose                                   |
| :------ | :--------- | :----------------- | :---------------------------------------- |
| `tbody` | None       | `tr`               | Defines the top-level element of a table. |
| `tr`    | None       | `th``td`           | Defines one row of a table.               |
| `td`    | None       | Any inline content | Defines one cell of a table row.          |
| `th`    | None       | Any inline content | Defines one header cell of a table row.   |



### 3.7 内联内容

| Element  | Attributes                  | Children       | Purpose                                                      |
| :------- | :-------------------------- | :------------- | :----------------------------------------------------------- |
| `a`      | `href` The URL of the link. | text           | Defines hyperlink. When a user selects the child text, they will be redirected to the given URL. |
| `b`      | None                        | Inline content | Defines content that should be displayed as bold face.       |
| `code`   | None                        | Inline content | Defines content representing code. It is typically shown in a monospace font. |
| `em`     | None                        | Inline content | Defines content that should be emphasized, typically by italicizing it. |
| `i`      | None                        | Inline content | Defines content that should be displayed as italics.         |
| `img`    | `src``alt``height``width`   | None           | Display an image. See the description above in Block elements. |
| `strong` | None                        | Inline content | Defines content that should be rendered more strongly, typically using bold face. |
| `sub`    | None                        | Inline content | Defines content that should be rendered as subscript.        |
| `sup`    | None                        | Inline content | Defines content that should be rendered as superscript.      |
| `tt`     | None                        | Inline content | Defines content that should be displayed with a monospace font. |



### 3.8 包含

为了在不同的帮助主题中使用重复的内容，可以将都用的内容存在查询帮助文件，其他文件帮助文件就可以使用`include`标签来引用。共享内容可以放在同一个目录或者放在 `SEMMLE_DIST/docs/include`。如果一个查询帮助文件只是被引用但不属于其他文件，需要使用`.inc.qhelp`后缀

 `include` 元素可以被用于Section-level 元素或者块元素。

**Section-level 包含元素**

Section-level 的`include`元素可以在`qhelp`中，例如

```
<qhelp>
    <include src="XSS.qhelp" />
</qhelp>
```

**Block-level 包含元素**

Block-level `include` 元素可以在section-level 元素里使用。例如，在`overview`中

```
qhelp>
    <overview>
        <include src="ThreadUnsafeICryptoTransformOverview.inc.qhelp" />
    </overview>
    ...
</qhelp>
```



## 4. 控制查询结果

可以通过select语句控制显示结果

### 4.1 关于查询结果

结果里的信息由`select`语句控制，`select`语句必须有两个字段

- Element——告警的变量，也就是要查询的代码
- String——自定义字符串消息

在有些查询中，我们可以看见除上面外的其他内容，使用 `$@` 做占位符。如下

```
select access, "Variable $@ may be null here " + msg + ".", var.getVariable(),
  var.getVariable().getName(), reason, "this"
```

后门的内容都会放在 `$@` 的位置。



### 4.2 编写select语句案例

`CodeDuplication.qll`是比较文件相似度的库。我们将在案例中使用这个库来辨析示例。

**基本的select语句**

```
import java
import external.CodeDuplication

from File f, File other, int percent
where similarFiles(f, other, percent)
select f, "This file is similar to another file."
```

这个基本查询语句的select语句有两个部分，

- `f`对应要对比的文件
- "This file is similar to another file."是字符串消息。

**在查询语句中显示其他文件信息**

上面的案例没有与其相似的文件信息，下面我们要打印出其他文件信息。

```
select f, "This file is similar to " + other.getBaseName()
```

这条语句可以将其他文件打印出来

**加入占位符**

```
select f, "This file is similar to $@.", other, other.getBaseName()
```

显示的时候other`, `other.getBaseName()`会替换`$@`的位置。

如果有多个占位符，原文如下，这里不深究

> If you use the `$@` placeholder marker multiple times in the description text, then the `N`th use is replaced by a link formed from columns `2N+2` and `2N+3`. If there are more pairs of additional columns than there are placeholder markers, then the trailing columns are ignored. Conversely, if there are fewer pairs of additional columns than there are placeholder markers, then the trailing markers are treated as normal text rather than placeholder markers.

**加入相似度信息**

```
select f, percent + "% of the lines in " + f.getBaseName() + " are similar to lines in $@.", other, other.getBaseName()
```



## 5. 元素位置信息

CodeQL提供了在代码数据库里获取元素位置信息的机制。使用这个机制可以为用户提供更多的信息。

### 5.1 关于元素位置

当显示给用户信息的时候，LGTM需要能够从查询结果中提取位置信息，为此，所有的QL 类提供位置信息需要使用如下机制之一

- 提供URLS
- 提供位置信息
- 使用提取的位置信息

上诉按优先级排列，以便使用第一个可用的机制。

> 因为QL是一种关系型语言，每一个QL类的实例都精确的映射到一个位置。这是库设计者的责任。如果实例没有位置信息，就不能点击结果跳转到源代码。如果有多个位置，结果就会重复。

### 5.2 提供URLS

一个自定义的URL可以写在一个没有参数的(注意大小写)的谓词getURL里，如下

```
class JiraIssue extends ExternalData {
    JiraIssue() {
        getDataPath() = "JiraIssues.csv"
    }

    string getKey() {
        result = getField(0)
    }

    string getURL() {
        result = "http://mycompany.com/jira/" + getKey()
    }
}
```

**文件 URLs**

`file://`跟一个绝对路径，后面再跟四个冒号分隔的数字。数字表示开始行、开始列、结束行和结束列。

- `file://opt/src/my/file.java:0:0:0:0` 表示使用整个文件
- `file:///opt/src/my/file.java:1:1:2:1` 从文件开始到第二行的第一个字符。
- `file:///opt/src/my/file.java:1:0:1:0` 表示文件的第一行

按照惯例，整个文件也可以用 `file://` 表示，后面不跟数字。或者使用三个数字来表示起始行的位置、长度、字符偏移量。LGTM不会显示这个类型的结果

**其他类型的URL**

以下不太常见的URL类型是有效的，但LGTM不支持，并且将从任何结果中忽略

- `HTTP URLs` 某些客户端应用支持。
- `Folder URLs` 用法： `folder://`,后面不跟数字，例如 `folder:///opt/src`
- `Relative file URLs` 像正常的文件URLs, 但是以 `relative://`开始。它们通常的意义是代码数据库的上下文。由于相对位置不变，导入外部信息的时候通常使Relative file URLs。



### 5.3 提供位置信息

如果没有定义getURL谓词，一个QL类会检查是否存在成员谓词 `hasLocationInfo(..)`。这可以理解为提供文件URLs的快捷方式而不用考虑写一个URL字符串。`hasLocationInfo(..)`的第一列必须是字符串类型（文件的URL路径），并且必须有3或4个int类型的字段，

例如，让我们想象一下，提取器提供的方法的位置从方法名称的第一个字符延伸到方法体的右大括号，我们想修改只提取方法名。下面的代码这两种的实现方式。

```
class MyMethod extends Method {
    // The locations from the database, which we want to modify.
    Location getLocation() { result = super.getLocation() }

    /* First member predicate: Construct a URL for the desired location. */
    string getURL() {
        exists(Location loc | loc = this.getLocation() |
            result = "file://" + loc.getFile().getFullName() +
                ":" + loc.getStartLine() +
                ":" + loc.getStartColumn() +
                ":" + loc.getStartLine() +
                ":" + (loc.getStartColumn() + getName().length() - 1)
        )
    }

    /* Second member predicate: Define hasLocationInfo. This will be more
       efficient (it avoids constructing long strings), and will
       only be used if getURL() is not defined. */
    predicate hasLocationInfo(string path, int sl, int sc, int el, int ec) {
        exists(Location loc | loc = this.getLocation() |
            path = loc.getFile().getFullName() and
            sl = loc.getStartLine() and
            sc = loc.getStartColumn() and
            el = sl and
            ec = sc + getName().length() - 1
        )
    }
}
```



### 5.4 使用提取的位置信息

如果上面的两个谓词失败了，客户端应用会尝试调用`getLocation()`的无参形式，并且会尝试上面的两个方法，拿到结果后存入数据库，指定表示符，以及使用

 `getLocation()` 返回的类称为`Location`，并且它会定义一个`hasLocationInfo(..)版（或者是`getURL()尽管前者更可取）。如果`Loacation` 类不提供任何成员谓词，就没有可用的位置信息。



### 5.5 toString谓词

除了扩展基类的所有类，必须提供`string toString()`成员谓词。如果不这样做编译器会告警。





## 6. 关于数据流分析

数据流分析用于计算一个变量在程序的各个点上可能保持的值，确定这些值如何在程序中传播以及在何处使用。

### 6.1 概述

许多CodeQL查询用于数据流分析，这个可以突出显示潜在的不安全数据的流动，这可能会产生漏洞。这些查询可以帮助你理解程序中的数据使用是否安全。是否将危险的参数传递给了函数，或者敏感数据是否会泄漏，以及强调显示潜在的安全问题，还可以使用数据流分析来了解程序行为的其他方面，通过发现，例如，未初始化的变量使用及资源泄漏。

接下来的部分会对CodeQL的数据流分析做一个简单的介绍。

更多针对特定语言数据流的分析如下地址：

- “[Analyzing data flow in C/C++](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-cpp/#analyzing-data-flow-in-cpp)”
- “[Analyzing data flow in C#](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-csharp/#analyzing-data-flow-in-csharp)”
- “[Analyzing data flow in Java](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-java/#analyzing-data-flow-in-java)”
- “[Analyzing data flow in JavaScript/TypeScript](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-javascript-and-typescript/#analyzing-data-flow-in-javascript-and-typescript)”
- “[Analyzing data flow in Python](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-python/#analyzing-data-flow-in-python)”

> 数据流分析一般用路径查询`path queries`



### 6.2 数据流图

CodeQL 数据流库通过对数据流图建模实现对程序的数据流分析。与抽象语法树不同，数据流图不能反映程序的语法结构图，但是对运行时数据流经程序的方式建模。抽象语法树中的节点表示语句或表达式等语法元素。数据流图中的节点，代表运行时语意元素的值。

一些AST([abstract syntax tree)节点（例如表达式）具有相应的数据流节点，但其他（比如if语句）则不然。这是因为表达式在运行时被计算成了一个值，而if语句纯粹是一个控制流构造，不携带值。也还有一些数据流节点根本不对应AST节点。

数据流图中的边（Edges）表示程序元素之间的数据流方式。例如，在表达式 `x || y` 中，数据流节点对应子表达式x和y，数据流节点也对于完整的表达式 `x || y` 。有一条边从节点x到节点`x || y`，代表数据可能从x流向`x || y`。同样的也有一条边从y到`x || y`。

局部（Local）数据流和全局（global）数据流的不同之处在于它们考虑的边：局部数据流只考虑属于同一函数的数据流节点之间的边，而忽略函数和对象属性之间的数据流。然而全局数据流也考虑后者。污点追踪引入了不和值流动完全对应的边。但要对运行时某些值是否可能衍生出其他值建模，例如通过一个字符串操作运行。

数据流图使用类对代表图节点的程序元素建模，节点之间的数据流使用谓词来计算边建模。

计算准确完整的数据流图带来了几个挑战：

- 在源代码不可用的情况下，无法通过标准库函数计算数据流。
- 有些行为直到运行时才确定，这意味着数据流库必须采取额外的步骤来找到潜在的调用目标。
- 变量之间的别名可能导致单次写入更改多个指针指向的值。
- 数据流图可能非常大，计算速度也很慢

为了客服这些潜在的问题，在库中对两种数据流进行了建模：

- 局部数据流(Local data flow)，关于单个函数中的数据流。在对本地数据流进行推理时。只用考虑同一个函数里两个节点之间的数据流。它通常足够快，对于许多查询，效率高且精确，它通常可以计算CodeQL数据库里所有的函数的局部数据流
- 全局数据流(Global data flow)，通过计算函数之间的数据流和对象属性，有效地考虑整个程序中的数据流。计算全局数据流通常比局部数据流更耗费时间和计算资源，因此尽可能将查询改进提升效率。

许多CodeQL查询包含局部和全局数据流，更多参考 [CodeQL query help](https://codeql.github.com/codeql-query-help)



### 6.3 正常数据流 vs 污点传播

在标准库，区分了正常数据流和污点传播。正常数据流库用于分析数据在程序中每一步流动的值。

例如，如果你追踪不安全的对象`x`(这可能是一些不可信或潜在的恶意数据)，程序中的某一步可能改变它的值。因此，在一个简单的处理中，例如 `y = x + 1`，一个正常的数据流分析会突出显示x而不是y。然而，因为y是x派生出来的，也是不可信的对象，因此y也被污染了，分析从x到y的污染称为污点传播。

在QL里，污点追踪通过包含不保留值的步骤来扩展正常数据流，但潜在的不安全对象任然在传播。污点传播库通过保存节点传播的谓词对传播过程建模。



## 7. 编写路径查询

你可以编写路径查询来呈现代码数据库中的数据流信息。

### 7.1 概述

安全研究员对程序中的数据流动方式尤其感兴趣。许多漏洞由正常数据流向意外的位置造成的。使用CodeQL编写的路径查询对于分析数据流特别有用，因为它们可以用来跟踪变量从可能的起点（source）到可能的终点（sink）的路径。要对路径建模，你的查询必须体统source和sink点的信息，以及连接它们的数据流步骤。

本主题提供有关如何构造路径查询文件的信息，以便您可以探索与数据流分析结果关联的路径。



### 7.2 构造路径查询

路径查询需要确定的元数据和查询谓词，和select语法结构。CodeQL中包含的许多内置边查询遵循一个简单的结构，这取决于你所分析的语言是如何用CodeQL建模的

你应该使用如下的模板

```
/**
 * ...
 * @kind path-problem
 * ...
 */

import <language>
// For some languages (Java/C++/Python) you need to explicitly import the data flow library, such as
// import semmle.code.java.dataflow.DataFlow
import DataFlow::PathGraph
...

from MyConfiguration config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "<message>"
```

- `DataFlow::Pathgraph`: 是需要从CodeQL标准库里导入的路径图模块
- `source` and `sink`是路径图的节点， `DataFlow::PathNode` 是它们的类型
- `MyConfiguration` 是一个包含了定义了怎样从source到sink数据流谓词的类。

### 7.3 路径查询的元数据

路径查询元数据必须包含 `@kind path-problem`，其他元数据取决于打算如何运行查询，



### 7.4 生成边解释

为了生成边解释，你的查询需要计算路径图。这需要你定义一个叫`edges`的查询谓词，这个谓词定义了正在计算的图的边关系。它用于计算与查询生成的每个结果相关的边（edge)，可以从一个标准数据流库中的路径图模块导入预定义的路径谓词。除了路径图模块，数据流库包含数据流分析中常用的其他类、谓词和模块。

```
import DataFlow::PathGraph
```

这条语句从数据流库(`DataFlow.qll`）导入了 `PathGraph`模块，这个模块定义了路径(`edges`)。CodeQL还包括很多额外的库，你还可以导入专门为在各种常见框架和环境中实现数据流分析而设计的库。查看数据流分析中使用的不同库的示例，查看更多关于标准库[standard libraries](https://codeql.github.com/codeql-standard-libraries)。

对于所有语言，还可以选择定义节点查询谓词，它指定您感兴趣的路径图的节点。如果节点被定义了，仅选择节点的路径。如果节点没有定义，选择所有可能的节点的路劲。

### 7.5 定义自己的路径(`edges`)谓词`predicate`

可以在自己编写的查询语句体力编写自己的路径谓词。按照如下格式

```
query predicate edges(PathNode a, PathNode b) {
/** Logical conditions which hold if `(a,b)` is an edge in the data flow graph */
}
```

更多内容在 [standard CodeQL libraries](https://codeql.github.com/codeql-standard-libraries) 搜索edges。



### 7.6 声明sources和sinks

在路径查询里面必须提供source和sink信息。这些对象与您正在探索的路径的节点相对应。source和sink的类型和名称必须在查询的from语句中定义，并且类型必须与路径谓词计算的图的节点兼容。

如果你正在查询C/C++, C#, Java, JavaScript, Python, 或者 Ruby 代码（在查询中使用了 `import DataFlow::PathGrap`代码），source和sink的定义由数据流库中的`Configuration`类定义。你应该在from语句中声明三个对象。例如：

```
from Configuration config, DataFlow::PathNode source, DataFlow::PathNode sink
```

通过导入数据流库来访问configuration类。这个类包含定义如何处理数据流的谓词。

- `isSource()` 定义数据可能从何而来
- `isSink()` 定义数据可能会流向哪里

你也可以通过扩展Configuration类来针对不同的框架环境定义自己的Configuration类，



### 7.7 定义数据流条件

where语句定义逻辑条件，where语句可以使用聚合、谓词和逻辑公式来使结果准确。在编写路径查询时，通常会包含一个谓词，该谓词仅在数据从source流向sink时有效。还可以通过使用`Configuration`配置的 `hasFlowPath` 谓词指定从source到sink的流。

```
where config.hasFlowPath(source, sink)
```


### 7.8 Select 语句

路径查询的select语句由四个字段组成

```
select element, source, sink, string
```

`element`和`source`分别代表告警的位置和告警的内容，`source`和`sink`是路径图中选择的节点。

`element`取决于你要查询的目的和类型，这对安全问题尤其重要。例如，如果你觉得`source`有问题，最好就显示`source`元素。相反，如果你觉得应该查看sink就应该选择sink元素显示



## 8. 提升查询性能

提升CodeQL查询语句的性能，遵循如下几条简单指导。

### 8.1 关于查询性能

本主题提供了一些关于如何避免可能影响查询性能的常见问题的简单提示，在阅读下面的提示之前，值得重申一下关于CodeQL和QL语言的重要观点。

- CodeQL的类和谓词都会生成数据库表，大型谓词生成包含许多行的大型表，因此计算成本很高。
- QL语言是使用标准数据库操作和关系代数实现的(例如join, projection, 和union)。有关查询语言和数据库的详细信息，看[About the QL language](https://codeql.github.com/docs/ql-language-reference/about-the-ql-language/#about-the-ql-language)
- 查询是自下而上计算的，这意味着一个谓词在它所依赖的所有谓词都被求值之前不会被求值。有关查询求值的详细信息，阅读[Evaluation of QL programs](https://codeql.github.com/docs/ql-language-reference/evaluation-of-ql-programs/#evaluation-of-ql-programs)



### 8.2 关于性能的tips

遵循以下指导原则，以确保您不会被最常见的CodeQL性能陷阱绊倒。

**消除笛卡尔积**

谓词的性能通常可以通过大致考虑它有多少个结果来判断。创建性能差的谓词的一种方法是使用两个变量，而不以任何方式将它们关联起来，或者只是用否定来联系他们。这将导致计算每个变量的可能值集之间的笛卡尔积，可能会生成一个庞大的结果表。如果不指定对变量的限制，可能会出现这种情况。例如，考虑以下谓词，检查java方法`m`是否可以访问字段`f`：

```
predicate mayAccess(Method m, Field f) {
  f.getAnAccess().getEnclosingCallable() = m
  or
  not exists(m.getBody())
}
```

如果m包含对f的访问，则谓词成立,但也保守地假设没有body的方法（例如，native方法）可以访问任何字段。

然而，如果m是native方法，那么mayAccess计算的表将包含代码库中所有字段f的一行`m，f`，这使得它可能非常大。

下面的示例展示一个更小的错误

```
class Foo extends Class {
  ...
  // BAD! Does not use ‘this’
  Method getToString() {
    result.getName() = "ToString"
  }
  ...
}
```

`getToString()`没有声明任何参数，但是有两个隐含参数result和this，并且没有说明。也就是说，每个叫ToString的方法或者Foo的实例，this都会在Foo类的getToString方法被约束。

**使用特殊的类型**

类型提供了关系大小的上线。这会有助于查询优化器更有效，因此，通常最好使用最具体的类型。例如：

```
predicate foo(LoggingCall e)
```

优先于

```
predicate foo(Expr e)
```

从类型上下文中，查询优化器推断出程序的某些部分是冗余的，并将其删除或专门化。

**确定变量的最具体类型**

如果您不熟悉查询中使用的库，可以使用CodeQL确定实体的类型。有一个称为 `getAQlClass()`的谓词，它会返回最具体的QL实体类型。

例如，如果您使用的是Java数据库，你可以在每一个`Expr`的调用使用 `getAQlClass()` 

```
import java

from Expr e, Callable c
where
    c.getDeclaringType().hasQualifiedName("my.namespace.name", "MyClass")
    and c.getName() = "c"
    and e.getEnclosingCallable() = c
select e, e.getAQlClass()
```

`getAQlClass()` 适合调试，真正运行的时候就不要用了，会影响性能

**避免复杂的递归**

您应该尽可能地简化递归谓词

**折叠谓词**

有时，您可以通过将大型谓词的部分“折叠”成较小的谓词来帮助查询优化器。

一般原则是将工作分成以下几部分：

- linear(线性)，这样就不会有太多分支。
- tightly bound，这样块就可以在尽可能多的变量上相互连接



## 9 Debugging





## 参考

- https://codeql.github.com/docs/codeql-overview/
- https://github.com/github/codeql/tree/main/docs

















