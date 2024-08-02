---
title: MySQL 连接问题总结
date: 2024-08-02 20:03:21
tags:
- DBMS
- MySQL
---

## 问题背景

在 `K8S` 集群上用 `bitnami/mysql` 部署了一套主从 `MySQL` 集群，提供给采购的 `3D` 标注平台使用。将后端服务扩容到 `3` 个实例后，`MySQL` 主节点频繁崩溃重启，日志只提示 `Forcing close of thread xx user: 'xxx'`。

## 问题排查

本来后端服务只有 `1` 个实例时，持续运行没有任何问题，但在扩容到 `3` 个实例后才开始暴露问题。有哪些指标会和实例数正相关呢？无非就是各种系统资源：

- CPU
- MEM
- Sockets

前两个指标可以直接排除，因为实例数增加并不会影响到 `MySQL` 集群占用的资源。`Sockets` 资源即连接数，通常在应用程序向数据库发起请求时打开，结束请求后关闭。一般地，应用程序为了避免频繁建立连接浪费太多系统资源，会建立连接池来保持多个长连接会话。这样当多个实例建立的最大连接数超过 `MySQL` 所允许的最大连接数时，新的连接请求会被拒绝（通常会返回一个错误信息，类似于 `ERROR 1040 (HY000): Too many connections`），甚至会导致我们所遇到的崩溃问题。

### 数据库连接配置

首先，我们来查看 `MySQL` 所允许的最大连接数：

```MySQL
SHOW VARIABLES LIKE 'max_connections';
```

默认情况下，`MySQL` 允许的最大连接数为 `151`：

```sql
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+

```

然后，我们再看查看 `MySQL` 当前已经建立的连接数，：

```sql
SHOW STATUS LIKE 'Threads_connected';
```

在我们的案例中，很明显看到当前连接数已经达到了最大，甚至在某一时刻跳到了 `152`：

```sql
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 151   |
+-------------------+-------+

```

这恰好能说明我们之前的猜想基本是正确的。

更进一步，我们可以查看当前已建立的连接详细信息：

```sql
SHOW PROCESSLIST;
```

结果如下：

```sql
mysql> SHOW PROCESSLIST;
+------+-----------------+--------------------+--------+-------------+-------+-----------------------------------------------------------------+------------------+
| Id   | User            | Host               | db     | Command     | Time  | State                                                           | Info             |
+------+-----------------+--------------------+--------+-------------+-------+-----------------------------------------------------------------+------------------+
|    5 | event_scheduler | localhost          | NULL   | Daemon      | 14053 | Waiting on empty queue                                          | NULL             |
|    6 | replicator      | 10.216.1.252:57170 | NULL   | Binlog Dump | 13979 | Source has sent all binlog to replica; waiting for more updates | NULL             |
|    7 | replicator      | 10.216.4.196:33804 | NULL   | Binlog Dump | 13880 | Source has sent all binlog to replica; waiting for more updates | NULL             |
|   16 | xxxxxx          | 10.216.0.2:50632   | anno3d | Sleep       |     2 |                                                                 | NULL             |
|   17 | xxxxxx          | 10.216.0.254:47908 | anno3d | Sleep       |     0 |                                                                 | NULL             |
|   18 | xxxxxx          | 10.216.0.254:37988 | anno3d | Sleep       |  1694 |                                                                 | NULL             |
|   19 | xxxxxx          | 10.216.0.2:33422   | anno3d | Sleep       |  1689 |                                                                 | NULL             |
...
+------+-----------------+--------------------+--------+-------------+-------+-----------------------------------------------------------------+------------------+
151 rows in set, 1 warning (0.01 sec)
```

每一行代表一个连接，可以根据需要管理和终止这些连接。

### 应用程序连接池配置

在我们的 `Spring Boot` 应用中，`application.properties` 文件中连接池配置默认如下：

```
spring:
  datasource:
    hikari:
      maximum-pool-size: 256
      minimum-idle: 64
```

这意味着在没有任何访问流量的情况下，单个实例也会和数据库建立 `64` 个长连接会话，而在我们的场景中，三个实例至少需要建立 `192` 个连接，而 `MySQL` 默认的 `151` 最大连接数显然是有问题的。

## 问题解决

找到了问题根源，我们就可以针对性地解决该问题。有如下两种解决方案：

- 调大 `MySQL` 最大连接数，通常是在 `my.cnf` 或 `my.cni` 文件中添加如下配置：

```
[mysqld]
max_connections=1024
```

- 调小 `Hikari` 连接数默认最小连接数（适用于业务流量较小的场景）：

```
spring:
  datasource:
    hikari:
      maximum-pool-size: 48
      minimum-idle: 12
```

## 总结

数据库一般需要配置 `max_connections` 来限制应用程序和其建立的长连接数，`MySQL` 默认是 `151`，而应用程序通常采用连接池来降低会话开销，这会导致空负载情况下也会和数据库建立一定数量的会话连接。当我们建立的应用程序实例数较多，而 `MySQL` 设置的 `max_connections` 较少时，可能会导致无法建立新的连接，因此导致应用程序无法正常使用，甚至会导致 `MySQL` 集群无法正常工作。
