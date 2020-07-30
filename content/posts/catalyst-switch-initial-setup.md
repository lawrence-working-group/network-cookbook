+++
title = "思科 Catalyst 系列交换机（IOS）入门配置指南"
date = "2020-07-30"
tags = ["Cisco", "IOS", "Catalyst"]
+++

# 重置系统配置

```
del flash:/vlan.dat
write erase
reload
```

如果系统提醒你配置没有保存，选择不要保存配置。

# 初始化设定

## 系统服务

```
no service config
service internal
service unsupported-transceiver
no ip http-server
no ip http secure-server
no spanning-tree vlan 1-4094
no errdisable detect cause gbic-invalid
vtp mode off
lldp run
cdp run
```

## 用户认证

```
enable secret <super-strong-enable-password>
aaa new-model
username <admin-user> privilege 15 secret <your-password>
aaa authentication login default local
aaa authorization exec default local
```

## 启用 SSH

（同时禁用 telnet）

```
hostname switch01
ip domain-name corp.example.com
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
line vty 0 4
 transport input ssh
```

## 启用 SNMP

如果你不使用 SNMP 就不要配置了。

```
snmp-server community 114514 RO
```

## DNS

```
ip name-server 8.8.8.8
```

## NTP

```
clock timezone CST 8 0
ntp server time.asia.apple.com prefer
```

# 二层交换配置

## 上行/傻瓜交换机端口

```
interface Gi1/0/1
 description uplink
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan all
 switchport trunk native vlan 1
 switchport mode trunk
```

## VLAN access 端口

```
interface range Fa1/0/1-20
 description computer
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
```

## LACP

```
port-channel load-balance src-dst-ip
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
interface GigabitEthernet0/51
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
interface GigabitEthernet0/52
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
```

# 三层交换/路由配置

## 简单三层管理 IP

```
interface vlan100
 ip address 192.168.1.100 255.255.255.0
ip default-gateway 192.168.1.1
```

## 跨 VLAN 路由

首先输入 `sdm prefer dual-ipv4-and-ipv6 routing` 然后 reload 一下。

```
system mtu routing 1500
ip routing
ip cef distributed
ipv6 cef distributed
interface vlan100
 ip address 192.168.1.100 255.255.255.0
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

## VRF 隔离管理 VLAN

```
vrf definition mgmt
 address-family ipv4
interface Vlan100
 vrf forwarding mgmt
 ip address 192.168.1.100 255.255.255.0
ip route vrf mgmt 0.0.0.0 0.0.0.0 192.168.1.1
```
