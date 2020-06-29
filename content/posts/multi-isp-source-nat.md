+++
title = "多 ISP 环境下的源地址 NAT/PAT 配置"
date = "2020-02-10"
tags = ["Cisco", "NAT", "route-map"]
+++

经常遇到多个一个路由器上有多个 ISP 的互联网访问，需要使得从不同接口流出的流量源地址进行地址转换（NAT）。
然而如果使用基于 access-list 的 `ip nat inside source` 选项，则只支持单个地址列表和单个互联网出口。
所以我们使用基于 route-map 的 `ip nat inside source route-map` 选项，能够为我们提供更大的灵活度。

本配置分为三部分，每一部分都依赖前一步的完成。

具体什么流量从什么出口离开路由器，取决于路由表以及策略路由的配置。

## 1. 配置一个匹配内网源地址的 access-list

以下示例匹配了所有 RFC 1918 定义的内网地址空间

```console
ip access-list standard rfc1918
 permit 10.0.0.0 0.255.255.255
 permit 172.0.0.0 0.31.255.255
 permit 192.168.0.0 0.0.255.255
 deny   any
```

## 2. 配置两个匹配内网源地址和互联网出口的 route-map

配置针对第一个 ISP 的 route-map，其出口为 GigabitEthernet1

```console
route-map NAT-toISP1 permit 10
 match ip address rfc1918
 match interface GigabitEthernet1
```

配置针对第二个 ISP 的 route-map，其出口为 GigabitEthernet2

```console
route-map NAT-toISP2 permit 10
 match ip address rfc1918
 match interface GigabitEthernet2
```

## 3. 配置针对两个出口的 NAT 地址池（可选）

```console
ip nat pool ISP1-POOL 192.0.2.10 192.0.2.20 prefix-length 24
ip nat pool ISP2-POOL 198.51.100.0 198.51.100.100 netmask 255.255.255.0
```

如果不配置 NAT 地址池，则可以在 NAT 策略中指定能够提供出方向的 NAT 源地址的网络接口名称

## 3. 配置两条 NAT 策略

```console
ip nat inside source route-map NAT-toISP1 pool ISP1-POOL overload
ip nat inside source route-map NAT-toISP2 pool ISP2-POOL overload
```

最后的 `overload` 关键词的含义是不仅仅进行地址转换，还要进行端口转换，即 PAT。
