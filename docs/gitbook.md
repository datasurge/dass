## gitbook

主要结构（一个简单的gitbook结构）：

```
.
├── book.json
├── README.md
├── SUMMARY.md
├── chapter-1/
|   ├── README.md
|   └── something.md
└── chapter-2/
    ├── README.md
    └── something.md
```

每个文件的作用：

| 文件          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `book.json`   | 存储[配置](https://toolchain.gitbook.com/config.html)数据（**可选**） |
| `README.md`   | 您的书的前言/介绍（**必填**）                                |
| `SUMMARY.md`  | 目录（参见[页数](https://toolchain.gitbook.com/pages.html)）（**可选**） |
| `GLOSSARY.md` | 词典/注释[术语](https://toolchain.gitbook.com/lexicon.html)列表（见[术语表](https://toolchain.gitbook.com/lexicon.html)）（**可选**） |

相对一个项目而言，项目和目录集成， 目录结构需要少许变动，并需要设置gitbook的目录相对位置：

```
.
├── book.json
└── docs/
    ├── README.md
    └── SUMMARY.md
```

设置在book.json中：

```
{
    "root": "./docs"
}
```

gitbook并支持忽略文件：`.gitignore`, `bookignore`以及`ignore`文件已获得文件和目录的跳过列表。

### 输出

1. gitbook可以把你的书本生成不同格式的电子书等， 并支持自定义和可扩展输出格式。

2. **默认**的风格，会将生成一个可交互的静态站点（html文件）
3. Portable Document Format (**PDF**) 是一以一种独立于软硬件，以及操作系统的方式来保存文档的格式。这是一种很普遍的格式。文件扩展为`.pdf`。
4. EPUB (electrontic publicaton的简称，有时称它为epub) 是一个由国际电子出版物论坛 (IDPF) 制定的免费并开放的电子书标准。文件扩展为`.epub`。
5. Mobipocket电子书格式是基于使用XHTML的开放电子书标准，并且可以包含JavaScript以及框架。

### 自述和介绍

Gitbook默认会在`README.md`文件中提取。如果在这个文件名没有出现在 `SUMMARY` 中，那么它会被添加为章节的第一个条目。

以及`github`中项目`README.md`一般问项目的概要叙述， 而不是使用文档的介绍以及开始，所以`gitbook`可以通过设置，将使用其他文件代替`README.md`。

只需要在`book.json`中添加一下配置：

```json
{
    "structure" : {
        "readme" : "myIntro.md"
    }
}
```



### 页面和摘要

主要使用SUMMARY.md文件来配置。 `SUMMARY.md`只是一个链接列表。链接的标题用作章节的标题，链接的目标是该章节文件的路径。将嵌套列表添加到父章节将创建子章节。

例子：

```markdown
# Summary

* [Part I](part1/README.md)
    * [Writing is nice](part1/writing.md)
    * [GitBook is nice](part1/gitbook.md)
* [Part II](part2/README.md)
    * [We love feedback](part2/feedback_please.md)
    * [Better tools for authors](part2/better_tools.md)
```

支持使用子章节，直接执行某一文件中的特定部分（Anchors）：

```markdown
# Summary

### Part I

* [Part I](part1/README.md)
    * [Writing is nice](part1/README.md#writing)
    * [GitBook is nice](part1/README.md#gitbook)
* [Part II](part2/README.md)
    * [We love feedback](part2/README.md#feedback)
    * [Better tools for authors](part2/README.md#tools)
```

支持使用`markdown`语法对标题或水平线样式进行分割：

```
# Summary

### Part I

* [Writing is nice](part1/writing.md)
* [GitBook is nice](part1/gitbook.md)

### Part II

* [We love feedback](part2/feedback_please.md)
* [Better tools for authors](part2/better_tools.md)

----

* [Last part without title](part3/title.md)
```



### 常用配置

不多赘述， [源网址](https://toolchain.gitbook.com/config.html)或[中文翻译版](https://chrisniael.gitbooks.io/gitbook-documentation/content/format/configuration.html)。

