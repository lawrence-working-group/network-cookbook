+++
title = "PPPoE 的 IPv6 前缀下发（Prefix-delegation）配置"
date = "2019-12-23"
tags = ["cisco", "IPv6", "PPPoE", "DHCPv6-PD"]
+++

在以太网环境下，DHCPv6-PD 比较容易配置和理解，而在 PPPoE 环境中，需要在 Dialer 接口配置中添加如下指令

```console
ipv6 enable
ipv6 dhcp client pd hint ::/56
ipv6 dhcp client pd <GENERAL_PREFIX_NAME>
```

其中 GENERAL_PREFIX_NAME 是一个自定义 IPv6 前缀的名字，从上游获得一个前缀之后，赋值给这个前缀名。
这个前缀名可以被分配给路由器的本地接口，但是不能以前缀名的形式分配添加路由。
本地接口使用 IPv6 General Prefix 的配置语法如下

```console
ipv6 address <GENERAL_PREFIX_NAME> 0:0:0:00A1::/64
```

在前述配置中，从上游获得前缀的长度是 56，而我们为接口分配的地址前缀长度往往是 64 位，所以需要给中间的 8 位一个确定的值。
上面一行配置中的 `00A1` 是故意这样写的（正常情况下的 IPv6 地址缩写为 `0:0:0:A1::/64`），其中 `A1` 即是接口 IPv6 地址需要明确的 8 位。
