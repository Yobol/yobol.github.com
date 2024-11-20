---
title: linux-shell-text-processing-tools-awk
date: 2024-11-21 00:33:46
tags:
- Linux
- Shell
- Text Processing
- Data Extration
- AWK
---
## 基本介绍

和 `sed` 与 `grep` 一样，`awk` 也有文本过滤功能，通常用于提取数据或生成报表，几乎所有的  `UNIX-like` 系统都自带这个工具。

`awk` 不仅仅是一个文本处理工具。为了高效解决问题，设计者们精心设计了一种特定领域语言（`Domain-Specific Language, DSL`)，使得编写文本处理脚本非常直观灵活（代价就是学习曲线非常陡峭）。

`awk` 可以依次处理输入文本的每一行，并读取每一个字节，特别适合处理 `LOG `、`CSV` 等具有相同格式的文本。

## 工作原理

## 基本用法

`awk` 基本用法：

```shell
awk [options] 'program' file
```

如输出 `/etc/passwd` 文件中所有用户名：

```shell
# -F 指定分隔符为 :, 表示每一行的字段之间用 : 分隔，默认是空格
# '{print $1}' 指定输出第一列：
#   '' 表示一个程序块
#   {} 表示对每一行执行的操作
#   print 是打印命令，$0 表示当前行，$1 表示第一列，$2 表示第二列，以此类推
# /etc/passwd 是输入文件
awk -F: '{print $1}' /etc/passwd
```

### 变量

除了提供 `${n}` 变量，`awk` 还提供了许多内置变量。

#### `NF`

`NF`：表示当前行共有多少字段，如：

```shell
awk -F: '{print NF}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行的字段数。

那么，如何输出每一行的最后一个字段呢？其实很简单，只需要输出第 `NF` 个字段即可：

```shell
awk -F: '{print $NF}' /etc/passwd
```

特别地，我们可以使用 `$(NF-1)` 来输出倒数第二个字段：

```shell
awk -F: '{print $(NF-1)}' /etc/passwd
```

#### `NR`

`NR`：表示当前行号，如：

```shell
awk -F: '{print NR}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行的行号。

特别地，我们可以使用如下命令输出每一行的行号和用户名：

```shell
awk -F: '{print NR ") " $1}' /etc/passwd
```

#### `FS`

`FS`：表示字段分隔符，默认是空格，如：

```shell
awk -F: '{print $1 FS $2}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行第一列和第二列，用 `:` 连接。

#### `RS`

`RS`：表示行分隔符，默认是换行符，如：

```shell
awk -F: '{print $1 RS $2}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行第一列和第二列，用换行符连接。

### 函数

### 条件

## 进阶使用

## 常见案例

## 参考

1. [Wikipedia | AWK](https://en.wikipedia.org/wiki/AWK)
