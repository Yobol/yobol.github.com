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

`NF`：表示当前行共有多少字段，即 `Number of Fields`，如：

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

`NR`：表示当前行在输入行中的下标，即 `Number of Records`，从 `1` 开始，如：

```shell
awk -F: '{print NR}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行的行号。

特别地，我们可以使用如下命令输出每一行的行号和用户名：

```shell
awk -F: '{print NR ") " $1}' /etc/passwd
```

#### `FS`

`FS`：表示字段分隔符，即 `Field Separator`，默认是空格，如：

```shell
awk -F: '{print $1 FS $2}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行第一列和第二列，用 `:` 连接。

#### `RS`

`RS`：表示行分隔符，即 `Record Separator`，默认是换行符，如：

```shell
awk -F: '{print $1 RS $2}' /etc/passwd
```

就表示输出 `/etc/passwd` 文件中每一行第一列和第二列，用换行符连接。

### 函数

> https://www.gnu.org/software/gawk/manual/html_node/Built_002din.html#Built_002din

`awk` 提供了许多内置函数，方便对原始数据进行处理。

#### `toupper()`

> https://www.gnu.org/software/gawk/manual/html_node/String-Functions.html#index-toupper_0028_0029-function

`toupper()`：表示将字符串转换为大写，如：

```shell
echo 'abc' | awk '{print toupper($0)}'
```

#### `tolower()`

> https://www.gnu.org/software/gawk/manual/html_node/String-Functions.html#index-tolower_0028_0029-function

`tolower()`：表示将字符串转换为小写，如：

```shell
echo 'ABC' | awk '{print tolower($0)}'
```

#### `length()`

> https://www.gnu.org/software/gawk/manual/html_node/String-Functions.html#index-length_0028_0029-function

`length()`：表示获取字符串长度，如：

```shell
echo 'abc' | awk '{print length($0)}'
```

#### `substr()`

> https://www.gnu.org/software/gawk/manual/html_node/String-Functions.html#index-substr_0028_0029-function

`substr()`：表示获取子字符串，首字母下标从 `1` 开始，如：

```shell
echo 'abc' | awk '{print substr($0, 0, 2)}'
ab

echo 'abc' | awk '{print substr($0, 1, 2)}'
ab

echo 'abc' | awk '{print substr($0, 2, 2)}'
bc
```

#### `rand()`

> https://www.gnu.org/software/gawk/manual/html_node/Numeric-Functions.html#index-rand_0028_0029-function

`rand()`：表示生成 `[0, 1)` 之间的随机数，如：

```shell
echo 'abc' | awk '{print rand()}'
```

### 条件

`awk` 提供了许多条件语句，只输出满足条件的行。

```shell
awk '[condition] <program>' file
```

#### 正则表达式

`awk` 支持正则表达式，如：

```shell
awk '/pattern/ <program>' file
```

我们可以选择输出 `/etc/passwd` 文件中所有包含 `/bin/bash` 的行：

```shell
awk '/\/bin\/bash/ {print $0}' /etc/passwd
```

#### 比较运算符

`awk` 支持比较运算符，如：

```shell
awk 'NR % 2 == 0 {print $0}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出偶数行。

```shell
awk 'NR > 10 {print $0}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出第 `10` 行之后的所有行。

#### 逻辑运算符

`awk` 支持逻辑运算符，如：

```shell
awk -F: '$1 == "root" || $1 == "yobol" {print $1}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出第一列是 `root` 或 `yobol` 的行。

### 语句

#### `if` 语句

`awk` 支持 `if` 语句，如：

```shell
awk -F: '{if ($1 > "o") {print $1} else {print "--"}}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出第一列大于 `o` 的行，否则输出 `--`。

#### `for` 语句

`awk` 支持 `for` 语句，如：

```shell
awk -F：'{for (i = 1; i <= NF; i++) {print $i}}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出每一行的每一个字段。

#### `while` 语句

`awk` 支持 `while` 语句，如：

```shell
awk -F: '{i = 1; while (i <= NF) {print $i; i++}}' /etc/passwd
```

表示从 `/etc/passwd` 文件中输出每一行的每一个字段。

## 进阶使用

## 常见案例

## 参考

1. [Wikipedia | AWK](https://en.wikipedia.org/wiki/AWK)
