+++
title = "Cisco IOS XE NAT 基础"
date = "2020-05-06"
tags = ["思科 Cisco", "IOS XE", "NAT"]
+++

开始之前，建议先把这些打上：

```
ip nat translation max-entries 2147483647
no ip nat service sip tcp port 5060
no ip nat service sip udp port 5060
```

## 理论知识

IOS XE 对 NAT 是按 flow 处理的，即如果进来的包源地址执行了某个变换操作，同一个 flow 的返回包的目标地址会被执行逆向的变换操作。因此，除特殊说明以外，下文只描述每个 flow 的第一个包的处理方法。

IOS XE 支持三种 NAT 模型：

* `inside source`：从 `inside` 接口收到的包的源 IP 从 inside local 地址换成 inside global 地址
* `outside source`：从 `outside` 接口收到的包的源 IP 从 outside global 地址换成 outside local 地址
* `inside destination`：从 `inside` 接口收到的包的目标 IP 从 inside global 地址换成 inside local 地址

其中 `inside destination` 入站方向只能按 access-list 选择 IP 地址，NAT 目标地址只能使用 ip pool，没有按协议或者端口选择的功能，因此 `inside destination` 方式绝大多数情况都用于三层负载均衡。

另外需要注意的一点是，从某接口收到的包意思就是这个包从某接口流入路由器，并非目标 IP 配置所在的接口。所以 Loopback 类型的接口往往是不用配置任何 `ip nat [inside|outside]` 语句的。

## 常见配置

### 内网用户共享单个公网 IP 地址

这是最常见的“家用路由器”配置，名字很多：NAT overload、source NAT (SNAT) masquerade、PAT 一般都是指这种情况。

NAT 选择 flow 的方法很多。我们可以用所有 IP 地址：
```
ip nat inside source list any4 interface GigabitEthernet0/0/0 [vrf <VRF 名称>] overload
ip access-list standard any4
 permit any
```

或者选择路由目标接口：
```
ip nat inside source route-map nat interface GigabitEthernet0/0/0 [vrf <VRF 名称>] overload
route-map nat permit 10 
 match interface GigabitEthernet0/0/0
```

`ip nat` 命令里面的 interface 上的第一个 IP 地址会被配置成 NAT 条目的 inside global 地址，这并不影响 NAT 的 flow 选择过程，这个端口也不一定要是上行端口。

如果你需要指定公网 IP 那头使用的端口范围，可以使用下列配置：
```
ip nat settings interface-overload port range start 1025 end 65535
ip nat settings interface-overload block port tcp 8080
ip nat settings interface-overload block port udp 5060
```

### 公网到内网的端口映射

这种在 Linux 上一般叫做 destination NAT。它的原理是，如果在公网地址的特定端口上收到了包，那么把进来的包的目标 IP 和目标端口分别换成内网特定主机的 IP 和特定端口。那么根据 flow 进出方向的包做逆向操作的规则，内网特定主机从特定端口发包被路由器收到以后，其源 IP 和源端口就要被换成公网 IP 和公网映射的端口。所以这里我们写一条 `inside source` 规则：
```
ip nat inside source static tcp <内网目标 IP> <内网目标端口> <公网 IP> <公网端口> [vrf <VRF 名称>] extendable
```

不过注意这种方法必须要内网设备返回的数据包经过当前路由器的 inside 端口。Linux 下面我们可以在入站方向 Destination NAT，出站方向 Source NAT (masquerade) 来实现类似应用层端口转发的效果；IOS XE 上就不能这么做了。

### 1:1 NAT

公网 IP 到内网 IP 的 1:1 映射。
```
ip nat inside source static <内网 IP> <公网 IP> [vrf <vrf 名称>] extendable 
```

### 三层负载均衡

```
ip access-list standard slb-inbound-ip permit 192.168.1.1
ip nat pool slb-destination 192.168.1.2 192.168.1.15 prefix-length 28 type rotary
ip nat inside destination list slb-inbound-ip pool slb-destination
```

### 跨 VRF 的 NAT

在同一 VRF 里面发生的 NAT，只需要在 NAT 规则后面加上 `vrf <VRF 名称>` 即可。

如果单侧 interface 在 VRF 里，或者其中一个 interface 启用了 MPLS，那么就需要配置 [Match-in-VRF 语句](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-3s/nat-xe-3s-book/iadnat-match-vrf.html)。

如果 NAT 的两侧 IP 分别在不同的 VRF 里面，那么必须使用 [VRF-Aware Software Infrastructure (VASI) 接口](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/200255-Configure-VRF-Aware-Software-Infrastruct.html)。VASI 接口的概念类似 JunOS 上的 `lt-0/0/0` 逻辑隧道接口。

## 故障排除

查看 NAT overload 的端口占用情况：
```
show ip nat portblock dynamic global detail
```

查看 NAT 表项：
```
show ip nat translations [verbose]
```

Debug NAT 操作：
```
debug ip nat <access-list>
```
