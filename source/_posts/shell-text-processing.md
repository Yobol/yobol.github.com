---
title: Shell 文本处理工具
date: 2024-08-15 00:11:56
tags:
- Shell
- Text Processing
- awk
- grep
- sed
---
## 综合示例

### `mc` 从远程批量下载本地缺失文件

`mc` 提供 `diff` 子命令用于对比 `SOURCE` 和 `TARGET` 之间的文件差异：

- `>` 表示文件只存在于 `TARGET`
- `<` 表示文件只存在于 `SOURCE`
- `!` 表示本地文件时间戳更新

因此，我们可以使用如下命令找出本地缺失的文件：

```shell
mc diff ./my-directory my-minio/my-bucket/my-directory | grep '>'
```

输出示例：

```shell
> https://s3.my-minio.cn/my-bucket/my-directory/a.zip
> https://s3.my-minio.cn/my-bucket/my-directory/b.zip
> https://s3.my-minio.cn/my-bucket/my-directory/c.zip
...
```

我们需要将 `https://s3.my-minio.cn` 替换成我们本地配置的 `alias: my-minio`。

首先，借助 `awk` 命令的 `-F` 选项来进行分离：`-F 'https://s3.my-minio.cn'` 用来指定使用 `'https://s3.my-minio.cn'` 分离 `'> https://s3.my-minio.cn/my-bucket/my-directory/a.zip'`，可以在 `'{print}'` 中分别使用 $1、$2 得到 `'> '`、`'/my-bucket/my-directory/a.zip'`，因为我们只取后半部分，所以使用 `$2`；然后，在 `'{print}'` 中使用 `"my-minio" $2` 的形式来拼接字符串。

示例如下：

```shell
mc diff ./my-directory my-minio/my-bucket/my-directory | grep '>' | awk -F 'https://s3.my-minio.cn' '{print "my-minio" $2}'
```

输出示例：

```shell
my-minio/my-bucket/my-directory/a.zip
my-minio/my-bucket/my-directory/b.zip
my-minio/my-bucket/my-directory/c.zip
```

最后，使用 `mc` 提供的 `cp` 命令将上述文件下载到本地：

```shell
# xargs 命令提供 -I replace-str 选项来指定占位符
mc diff ./my-directory my-minio/my-bucket/my-directory | grep '>' | awk -F 'https://s3.my-minio.cn' '{print "my-minio" $2}' | xargs -I {} mc cp {} .
```
