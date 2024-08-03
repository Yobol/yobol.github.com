---
title: netfilter 框架初识之 iptables 工具使用上手
date: 2024-07-28 19:02:41
tags:
- Linux
- Network
- netfilter
- iptables
- Packet Filtering Software
- NAT
---
# netfilter 框架初识

`netfilter` 项目是一个由社区驱动的协同自由开源软件（`FOSS`，`Free and Open-source Software`）项目，为 `Linux 2.4.x` 及更高版本的内核系列提供**数据包过滤软件**（`packet filtering software`），如 `iptables` 以及其后继者 `nftables`。

> Linux 2.4.x 支持了 ip_tables 数据包过滤特性

`netfilter` 项目提供**数据包过滤**、**网络地址（和端口）转换（NA[P]T）**、**数据包记录**、**用户空间数据包队列**以及其它数据包处理功能。

`netfilter` 提供了一套 `Hooks` 框架，使得 `Linux` 内核模块可以在 `Linux` 网络栈的不同位置注册回调函数，当数据包经过这些 `hook` 时，注册的回调函数会依次被调用。

`iptables` 是一个通用的防火墙软件，允许自定义规则集。`IP` 表中的每条规则都由若干分类项（`iptables matches`）和一个相关联的动作（`iptables target`）组成。

`nftables`，即下一代防火墙（`next-generation firewall in Linux`），旨在取代 `iptables`，提供更灵活、可扩展和高性能的数据包分类功能。

## 主要特性

- 无状态的数据包过滤（`IPv4` 和 `IPv6`），仅允许特定 `IP` 地址的流量进入，无需检查流量连接状态；
- 有状态的数据包过滤（`IPv4` 和 `IPv6`），允许已建立或相关的连接通过，而新连接需要符合特定规则；
- 各种网络地址和端口转换，如 `NAT/NAPT`（`IPv4` 和 `IPv6`），即提供 `NA[P]T` 功能以允许通过设置规则来进行网络地址转换，如将外部请求的 `IP` 地址转换为内部服务器的 `IP` 地址（源地址转换，`SNAT`）或者将多个内部服务器的 `IP` 地址映射到单个外部 `IP`（端口地址转换，`NAPT`）；
- 灵活且可扩展的基础设施，即提供模块化的架构以允许开发者们根据需求自定义模块来处理数据包；
- 用于第三方扩展的多层 API 接口，即提供 `Hooks` 框架以允许第三方程序在 `Linux` 网络栈不同的处理阶段插入自定义处理逻辑，如监控网络流量或实现特定安全策略。

## 应用场景

- 基于有/无状态的数据包过滤来构建防火墙；
- 部署高可用的有/无状态防火墙集群；
- 如果没有足够的公网 `IP` 地址，可以使用 `NAT `和 `Masquerading` 功能来共享互联网连接；
- 使用 `NAT` 实现透明代理；
- 帮助 `tc` 和 `iproute2` 系统构建复杂的 `QoS` 和策略路由；
- 做进一步的数据包处理（`mangling`），例如概念 `IP` 头中的 `TOS/DSCP/ECN` 位。

# iptables 工具初识

`iptables` 是一个用户空间命令行工具，用于为 `Linux 2.4.x` 及更高版本的内核系列**配置数据包过滤规则集**，主要用户是操作系统管理员等服务器运维人员。

由于 `NAT` 也是基于数据包过滤规则进行配置的，因此 `iptables` 也可以用于 `NA[P]T` 配置。

`iptables` 用于 `IPv4` 数据包过滤，而 `ip6tables` 则用于 `IPv6` 数据包过滤。特别地，`nftables` 统一了这两个以及 `{eb,arp}tables` 工具，可以使用一致的语法进行配置。

## 主要特性

- 列出所有数据包过滤规则集的内容
- 在数据包过滤规则集中增加/删除/修改规则；
- 列出/清零数据包过滤规则集中每条规则的计数器。

# iptables 工具上手

# 总结

# 参考

1. [netfilter.org](https://www.netfilter.org/)
