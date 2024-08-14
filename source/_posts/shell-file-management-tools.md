---
title: Shell 文件管理工具集
date: 2024-08-15 01:39:07
tags:
- Shell
- File Management
---
## 原理解析

## 工具介绍

### ls

`ls` 命令，即 `list`，用于查看目录中的文件及其属性信息。

#### 常用参数

| 可选项 | 说明 |
| ----  | ---- |
| -a | 显示所有文件及目录（包含所有隐藏文件） |
| -A | 不显示当前目录和父目录 |
| -d | 只显示当前目录自身信息 |
| -i | 显示文件的 inode 信息 |
| -l | 显示文件的详细信息 |
| -h | 以可读性更好的形式显示文件大小 |
| -r | 按照文件名称降序排列 |
| -S | 按照文件大小降序排列 |
| -t | 按照修改时间降序排列 |
| -X | 按照文件后缀升序排列 |

#### 使用示例

##### 显示当前目录下的所有文件名（不包含隐藏文件）

```shell
ls
```

输出示例：

```shell
d1 d2 d3 f1 f2 f3
```

##### 显示当前目录下的所有文件名（包含隐藏文件）

```shell
ls -a
```

输出示例：

```shell
. .. d1 d2 d3 f1 f2 f3
```

##### 详细显示当前目录下的文件及属性信息（不包含 inode  信息）

```shell
ls -al
```

输出示例：

```shell
total 20
drwxr-xr-x  5 yobol yobol 4096  8月 15 01:53 .
drwxr-xr-x 21 root  root  4096  8月 15 01:47 ..
drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d1
drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d2
drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d3
-rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f1
-rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f2
-rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f3
```

##### 详细显示当前目录下的文件及属性信息（包含 inode 信息）

```shell
ls -ali
```

输出示例：

```shell
total 20
14417921 drwxr-xr-x  5 yobol yobol 4096  8月 15 01:53 .
       2 drwxr-xr-x 21 root  root  4096  8月 15 01:47 ..
14417922 drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d1
14417923 drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d2
14417924 drwxrwxr-x  2 yobol yobol 4096  8月 15 01:53 d3
14417925 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f1
14417926 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f2
14417927 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f3
```

##### 以可读性更好的形式显式文件大小

```shell
ls -alih
```

输出示例：

```shell
total 20K
14417921 drwxr-xr-x  5 yobol yobol 4.0K  8月 15 01:53 .
       2 drwxr-xr-x 21 root  root  4.0K  8月 15 01:47 ..
14417922 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d1
14417923 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d2
14417924 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d3
14417925 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f1
14417926 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f2
14417927 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f3
```

##### 按照文件大小降序排列

```shell
ls -aliht
```

输出示例：

```shell
total 20K
14417921 drwxr-xr-x  5 yobol yobol 4.0K  8月 15 01:53 .
14417925 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f1
14417926 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f2
14417927 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f3
14417922 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d1
14417923 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d2
14417924 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d3
       2 drwxr-xr-x 21 root  root  4.0K  8月 15 01:47 ..
```

##### 按照修改时间降序排列

```shell
ls -alihS
```

输出示例：

```shell
total 20K
14417921 drwxr-xr-x  5 yobol yobol 4.0K  8月 15 01:53 .
       2 drwxr-xr-x 21 root  root  4.0K  8月 15 01:47 ..
14417922 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d1
14417923 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d2
14417924 drwxrwxr-x  2 yobol yobol 4.0K  8月 15 01:53 d3
14417925 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f1
14417926 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f2
14417927 -rw-rw-r--  1 yobol yobol    0  8月 15 01:53 f3
```

##### 显式指定目录下的文件详情

```shell
ls -alih ./d2
```

输出示例：

```shell
total 8.0K
14417923 drwxrwxr-x 2 yobol yobol 4.0K  8月 15 01:53 .
14417921 drwxr-xr-x 6 yobol yobol 4.0K  8月 15 02:02 ..
```

##### 结合通配符显式指定文件详情

```shell
ls -alih ./d*
```

输出示例：

```shell
./d1:
total 8.0K
14417922 drwxrwxr-x 2 yobol yobol 4.0K  8月 15 01:53 .
14417921 drwxr-xr-x 6 yobol yobol 4.0K  8月 15 02:02 ..

./d2:
total 8.0K
14417923 drwxrwxr-x 2 yobol yobol 4.0K  8月 15 01:53 .
14417921 drwxr-xr-x 6 yobol yobol 4.0K  8月 15 02:02 ..

./d3:
total 8.0K
14417924 drwxrwxr-x 2 yobol yobol 4.0K  8月 15 01:53 .
14417921 drwxr-xr-x 6 yobol yobol 4.0K  8月 15 02:02 ..
```
