+++
title = "OSPF IPv4 Unnumbered 在以太网环境中的配置"
date = "2023-12-17"
tags = ["H3C", "OSPF", "Unnumbered"]
+++

## 先说结论
可以在点对点以太网环境中，配置基于 IP Unnumbered OSPF，可以达到节约 IPv4 地址、简化配置、即插即用的目的。
而 IPv6 场景中路由协议会充分利用接口上的 Link-local 地址，实现简化配置的效果。

## IP Unnumbered 介绍
IP Unnumbered 是一种允许在没有分配 IP 地址的接口上运行 IP 的技术。这种技术最初是为了解决在串行链路（如点对点连接）上 IP 地址短缺的问题。在这种情况下，每个串行接口都需要一个唯一的 IP 地址，这会消耗大量的 IP 地址。

IP Unnumbered 技术通过借用已分配给其他接口的 IP 地址，使得没有分配 IP 地址的接口也能运行 IP。这样，就可以在不消耗额外 IP 地址的情况下，使得更多的接口能够运行 IP。

即使不显式在接口上配置 IP 地址，当网络设备接口上启用 IP Unnumbered 时，也会开启在这个接口上的 IP 包处理。

## OSPF Unnumbered 适用场景及优点
目前（2023 年）串行链路已经比较少见，企业中最常见的连接介质是以太网。而在以太网上配置 OSPF，最常见的教科书级的配置模式有两种：

1. 广播（Broadcast）：这种类型适用于可以发送广播数据包的网络，如以太网。在这种网络类型中，OSPF 使用多播地址发送 Hello 包，通过接收 Hello 包来发现邻居。广播环境中的 OSPF 需要选举 DR 和 BDR。

1. 点对点（Point-to-Point）：这种类型适用于只有两个端点的网络，如串行链路，或者两个三层口直接互联的以太网接口设备。在这种网络类型中，OSPF 使用多播地址发送 Hello 包，通过接收 Hello 包来发现邻居。点对点环境中 OSPF 无需选举 DR 或 BDR，能够减少邻接关系建立的时间。

以太网环境中默认的 OSPF 配置，需要在两侧的接口上共配置两个在同一个子网的 IPv4 地址，这通常意味着要使用一个 /30 大小的子网，虽然现在主流网络设备已经支持了 /31 作为三层互联地址（RFC3021），但是总会有一些软件尚不支持。如此，就意味着每两个设备就要使用 2 个或 4 个 IPv4 地址。而当使用了 IP Unnumbered 后，每个设备无论多少 OSPF 邻接关系（只要是点对点），只需要使用 1 个 IPv4 地址即可。

## OSPF Unnumbered 在不同厂商设备上的配置

### Cisco IOS/IOS XE 上的配置
TODO

### Juniper Junos 上的配置
TODO

### H3C Comware V7 上的配置
#### 全局 OSPF 配置
```
ospf 100
 silent-interface all
 undo silent-interface FortyGigE1/0/25
 undo silent-interface FortyGigE1/0/26
 undo rfc1583 compatible
 bandwidth-reference 100000
#
```
释义：默认所有接口不处理 OSPF 报文；只在两个 40G 接口（FGE1/0/25 和 FGE1/0/26）收发 OSPF 报文；去掉对已废除 RFC1583 选路策略的兼容性；接口开销带宽参考值设置为 100x1000Mbps=100Gbps。
#### 接口相关配置
```
interface LoopBack1
 description RouterID and PeeringIP
 ip address 172.30.1.1 255.255.255.255
#
interface FortyGigE1/0/25
 port link-mode route
 ip address unnumbered interface LoopBack1
 ospf network-type p2p
 ospf 100 area 0.0.0.0
 lldp management-address arp-learning
 lldp tlv-enable basic-tlv management-address-tlv interface LoopBack1
#
```
释义：配置一个 Loopback1 接口为 IP unnumbered 提供地址来源；配置三层接口，配置三层接口的 OSPF 网络类型为点对点（P2P）；配置 OSPF 区域；配置通过 LLDP 学习对方的 Loopback 1 上的接口地址，并写入自身 ARP 表。

__经过各种尝试，虽然这种通过 LLDP 学习 ARP 条目的方式有悖网络不同层次之间交互的一般逻辑，但是的确是 H3C Comware V7 平台上唯一好用的方法。__

在链路两侧的设备上完成配置后，除了 Loopback 1 的接口地址不同，其他配置都是相同的，可以建立起 OSPF 邻接关系，相互传递链路状态路由。

在这篇文章[3]中指出 Comware V7 平台的 `ospf network-type p2p` 会以单播形式发送报文，通过查阅 H3C 命令手册[4]，可以看到，如果配置为 `ospf network-type p2p multicast` 就会在点对点链路上继续向组播地址发送报文。但是即使通过修改这个参数可以实现路由信息的交换，但是由于没有对方的 ARP 条目，依然无法实现数据包转发。究其原因，是此平台上，似乎是不支持 IP Unnumbered 接口学习对方发来的 ARP 响应信息。

## 参考资料
1. [RFC3021](https://datatracker.ietf.org/doc/html/rfc3021) Using 31-Bit Prefixes on IPv4 Point-to-Point Links: https://datatracker.ietf.org/doc/html/rfc3021
1. [RFC2328](https://www.rfc-editor.org/rfc/rfc2328) OSPF Version 2: https://www.rfc-editor.org/rfc/rfc2328 （最初版本的 OSPFv2 在 RFC 1583中定义，现已被废除）
1. 《【新华三】ospf借用地址建立邻居停留在Exstart状态分析 ( CSDN 博客)》: https://blog.csdn.net/weixin_41563735/article/details/131756415
1. 《05-三层技术-IP路由命令参考》: https://www.h3c.com/cn/d_202310/1948070_30005_0.htm#_Toc148032918
