+++
title = "交换机上的以太网链路聚合配置"
date = "2020-05-19"
tags = ["新华三 H3C", "交换机", "链路聚合", "LACP", "瞻博 Juniper", "思科 Cisco", "Junos", "IOS"]
+++

本文只介绍 LACP 的聚合方式，如无特殊要求，不应当使用静态聚合。因为静态聚合配置下，接口是否在聚合组中被选中依赖的是接口的 line protocol 状态，在连接了一些不识别不处理 LACP 数据包的网络设备时，设备两侧的接口状态可能不同步，造成部分链路的流量被丢弃。

LACP 控制报文最快一秒钟发送一次，此时最长需要三秒钟才能检测到链路故障。在一些要求链路状态变化时更迅速调整以减少丢包的场景下，需要配置 BFD，减少故障检测所需的时间。

## 新华三交换机
本章节是以新华三运行 Comware V7 R1312 的 S5560-EI 交换机为基础编写的。如果你使用的交换机操作系统版本过于陈旧，部分命令可能并不适用。

### 链路聚合模式
与思科交换机支持三种链路聚合模式（静态、私有 PAgP、LACP）相似，新华三交换机支持静态聚合和基于 LACP 的动态聚合。参考[《H3C 技术甜甜圈 - 链路聚合协议互通测试》](http://www.h3c.com/cn/d_201405/828443_97665_0.htm)。

### 动态链路聚合（LACP）的配置
新华三 Comware V7 平台上，两种聚合接口在配置步骤上是一致的：

1. 创建一个链路聚合逻辑接口
2. 对链路聚合逻辑接口的相关属性进行配置
3. 进入（二三层对应的）交换机物理接口配置视图，将物理接口添加为链路聚合逻辑接口的成员
4. 配置物理接口上的链路聚合相关属性
5. 检查链路聚合逻辑接口的状态和链路聚合的配置结果

#### 二层聚合 Bridge-Aggregation
```
interface bridge-aggregation <聚合逻辑口编号>
 link-aggregation mode dynamic

interface <物理接口类型><物理接口编号>
 port link-aggregation group <聚合逻辑口编号>
 lacp period short
```

#### 三层聚合 Route-Aggregation
```
interface route-aggregation <聚合逻辑口编号>
 link-aggregation mode dynamic

interface <物理接口类型><物理接口编号>
 port link-aggregation group <聚合逻辑口编号>
 lacp period short
```

#### 聚合边缘接口
LACP 边缘接口的使用场景：交换机与服务器等终端设备相连，终端设备的生命周期中，存在未启用 LACP 但是要求网络通信的时刻，例如未配置的 [VMware vSphere Distributed Switch (VDS)](https://www.vmware.com/products/vsphere/distributed-switch.html)，以及通过 PXE 启动的设备。

- 在二层聚合接口
  ```
  interface bridge-aggregation <聚合逻辑口编号>
   lacp edge-port
  quit
  ```
- 在三层聚合接口
  ```
  interface route-aggregation <聚合逻辑口编号>
   lacp edge-port
  quit
  ```

#### 检查验证
1. `display link-aggregation summary` 命令用来显示所有聚合组的摘要信息。
1. `display link-aggregation verbose` 命令用来显示已有聚合接口所对应聚合组的详细信息。
1. `display link-aggregation member-port <物理接口类型和编号>` 命令用来查询物理端口属于哪个链路聚合组的的详细信息。

## Cisco IOS

```
interface GigabitEthernet0/51
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
interface GigabitEthernet0/52
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 speed nonegotiate
```

## Juniper Junos

首先我们创建出所需数量的链路聚合虚拟端口（`aeN`），并把相应的物理端口加入相应的虚拟端口组：

```
set chassis aggregated-devices ethernet device-count 1
delete interfaces ge-0/0/1
delete interfaces ge-0/0/2
set interfaces ge-0/0/1 gigether-options 802.3ad ae0
set interfaces ge-0/0/2 gigether-options 802.3ad ae0
```

然后设置 LACP 参数：

```
# set interfaces ae0 mtu 1522
set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 aggregated-ether-options lacp periodic fast
set interfaces ae0.0 vlan-id 1
```

注意 JunOS 在 `aeN` 没有配置 `unit 0` 的情况下不会启动 LACP 握手，端口会始终显示为 down。

### 参考资料
- [《H3C - 以太网链路聚合配置》](https://www.h3c.com/cn/d_201912/1252416_30005_0.htm)
- [《H3C - 以太网链路聚合命令》](https://www.h3c.com/cn/d_201912/1252029_30005_0.htm)
- [Cisco - Link Aggregation Control Protocol (LACP) (802.3ad) for Gigabit Interfaces](https://www.cisco.com/c/en/us/td/docs/ios/12_2sb/feature/guide/gigeth.html)
- [Cisco - Catalyst 6500 Release 15.0SY Software Configuration Guide - EtherChannels](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/15-0SY/configuration/guide/15_0_sy_swcg/etherchannel.html)
- [Juniper - TechLibrary - Junos OS - Aggregated Ethernet Interfaces](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/switches-interface-aggregated.html)
