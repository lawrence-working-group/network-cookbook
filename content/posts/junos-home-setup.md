+++
title = "Juniper SRX 家用路由器配置"
date = "2020-05-29"
tags = ["JunOS"]
+++

# 恢复出厂设置

如果登录不了，可能需要 [重置 root 密码](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/recovering-root-password.html) 或者从 loader 安装系统以格式化硬盘。

进入系统以后，使用

```
request system zeroize
```

恢复出厂设置。

# 设置 root 密码

设备启动以后，使用默认用户名 `root` 空密码登录，然后：

```
root@Amnestic% cli
root@Amnestic> configure
Entering configuration mode

[edit]
root@Amnestic# set system root-authentication plain-text-password
New password: # 输入新密码
Retype new password: # 重复输入新密码
root@Amnestic# commit
```

# 家用路由器配置

## 启用 IPv6

```
set security forwarding-options family inet6 mode flow-based
```

commit 完重启一下。

## 基础系统设置

```
deactivate system phone-home
set system syslog file messages match "!(.*usage requires a license.*)"
set system services ssh
del system services netconf ssh
del system services web-management
set protocols l2-learning global-mode switching

# 主机名
set system host-name srx114514

# DNS 服务器
set system name-server 8.8.8.8
set system name-server 8.8.4.4

# NTP 服务器
set system ntp server time.asia.apple.com prefer
set system ntp server cn.pool.ntp.org
```

## 简单的分区防火墙

有些型号上这些配置是自带的，有些不是，如果原来就有就不必要再配置一遍了。

```
set security policies from-zone trust to-zone trust policy trust-to-trust match source-address any
set security policies from-zone trust to-zone trust policy trust-to-trust match destination-address any
set security policies from-zone trust to-zone trust policy trust-to-trust match application any
set security policies from-zone trust to-zone trust policy trust-to-trust then permit
set security policies from-zone trust to-zone untrust policy trust-to-untrust match source-address any
set security policies from-zone trust to-zone untrust policy trust-to-untrust match destination-address any
set security policies from-zone trust to-zone untrust policy trust-to-untrust match application any
set security policies from-zone trust to-zone untrust policy trust-to-untrust then permit
set security zones security-zone trust host-inbound-traffic system-services all
set security zones security-zone trust host-inbound-traffic protocols all
set security zones security-zone untrust screen untrust-screen
```

## LAN 设置

```
# VLAN 设置
set interfaces irb unit 100 family inet address 192.168.1.1/24
set vlans vlan-trust vlan-id 100
set vlans vlan-trust l3-interface irb.100
set security zones security-zone trust interfaces irb.100

# Access 接口设置
set interfaces interface-range lan member "ge-0/0/[1-7]"
set interfaces interface-range lan unit 0 family ethernet-switching interface-mode access
set interfaces interface-range lan unit 0 family ethernet-switching vlan members vlan-trust

# 本机 DNS 服务器
set system services dns dns-proxy interface irb.100

# DHCP 服务器
set access address-assignment pool lan family inet network 192.168.1.1/24
set access address-assignment pool lan family inet range r1 low 192.168.1.2
set access address-assignment pool lan family inet range r1 high 192.168.1.254
set access address-assignment pool lan family inet dhcp-attributes router 192.168.1.1
set access address-assignment pool lan family inet dhcp-attributes name-server 192.168.1.1
set access address-assignment pool lan family inet dhcp-attributes propagate-settings pp0.0
set system services dhcp-local-server group lan interface irb.100

# NAT Masquerade
set security nat source rule-set trust-to-untrust from zone trust
set security nat source rule-set trust-to-untrust to zone untrust
set security nat source rule-set trust-to-untrust rule source-nat-rule match source-address 0.0.0.0/0
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface

# NAT 类型（三个当中选一个，不写的话默认 symmetric）
## Full Cone （注意和端口映射不同，这里的 Full Cone 仍然是动态生成的规则）
set security nat source interface port-overloading off
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat permit any-remote-host
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat max-session-number 65536
## Restricted Cone
set security nat source interface port-overloading off
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat permit target-host
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat max-session-number 65536
## Port-restricted Cone
set security nat source interface port-overloading off
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat permit target-host-port
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat max-session-number 65536
```

## WAN 设置

### PPPoE 上联

```
# 拨号和 IPv4
delete interfaces ge-0/0/0.0 family ethernet-switching
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 native-vlan-id 1
set interfaces ge-0/0/0 unit 0 encapsulation ppp-over-ether
set interfaces ge-0/0/0 unit 0 vlan-id 1
set interfaces pp0 unit 0 description "ChinaNet uplink"
set interfaces pp0 unit 0 ppp-options pap local-name your-username
set interfaces pp0 unit 0 ppp-options pap local-password your-password
set interfaces pp0 unit 0 ppp-options pap passive
set interfaces pp0 unit 0 pppoe-options underlying-interface ge-0/0/0.0
set interfaces pp0 unit 0 pppoe-options idle-timeout 0
set interfaces pp0 unit 0 pppoe-options auto-reconnect 5
set interfaces pp0 unit 0 pppoe-options client
set interfaces pp0 unit 0 pppoe-options ignore-eol-tag
set interfaces pp0 unit 0 family inet mtu 1492
set interfaces pp0 unit 0 family inet negotiate-address
set routing-options static route 0.0.0.0/0 next-hop pp0.0
# IPv4 Only
#set security flow tcp-mss all-tcp mss 1452
# IPv4 and IPv6
set security flow tcp-mss all-tcp mss 1432
set security zones security-zone untrust interfaces ge-0/0/0.0
set security zones security-zone untrust interfaces pp0.0

# IPv6
# 以下配置是DHCPv6 PD模式的，如果你这样配置后发现运营商没下发PD, 请看后文IPV6注解
set security zones security-zone untrust host-inbound-traffic system-services dhcpv6
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-type stateful
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-ia-type ia-pd
set interfaces pp0 unit 0 family inet6 dhcpv6-client rapid-commit
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-identifier duid-type duid-ll
set interfaces pp0 unit 0 family inet6 dhcpv6-client req-option dns-server
set interfaces pp0 unit 0 family inet6 dhcpv6-client retransmission-attempt 5
set interfaces pp0 unit 0 family inet6 dhcpv6-client prefix-delegating preferred-prefix-length 56
set interfaces pp0 unit 0 family inet6 dhcpv6-client prefix-delegating sub-prefix-length 64
set interfaces pp0 unit 0 family inet6 dhcpv6-client update-router-advertisement interface irb.100 other-stateful-configuration
set interfaces pp0 unit 0 family inet6 dhcpv6-client update-router-advertisement interface irb.100 max-advertisement-interval 180
set routing-options rib inet6.0 static route ::/0 next-hop pp0.0
```

## 保存备份配置

一切设置妥当以后，保存一份配置以便在故障恢复时使用。

```
request system configuration rescue save
request system autorecovery state save
```

# IPV6 注解

## tcp-mss 值注意事项

在 IPV6 场景下，mss 值要从通常的 1452 减去 20 个字节变成 1432 .

之前 IPV6 环境下访问某些网站，时而可以访问时而不能访问的根本原因找到了，原因是 PMTU 黑洞，其细节如下：

终端设备在发包时，也可以设置 DF （ Don't Fragment ）标记来告诉路由器不要分片。这时中间路由器会丢掉超过 MTU 的包，回复一条 ICMP Fragmentation Needed 消息。发送者收到这个包后，下次就会发小一点的包，这个过程叫做 PMTU Discovery . 现实中可以看到 HTTPS （ TLS ）的流量大都是带 DF 标记的。
然而，互联网上有大量的中间设备为了所谓的“安全”或者没有正确配置，不回应 ICMP Fragmentation Needed 包，这使得访问某些网站时如果某个包的大小超过了 PMTU ，会被无声地丢弃，直到 TCP 协议发现超时丢包进行重传，这非常缓慢。遇到这种情况，我们可以说你和目标服务器的路径上存在 PMTU 黑洞。
由于我们到某些站点之间的目标链路一直在变化，在链路节点中如果遇到了这种被错误配置的设备，就会导致我们无法访问这些站点。
现在国内 ISP 一般都是通过 PPPoE 虚拟拨号建立 WAN 口连接的。Ethernet 的默认 MTU 是 1500，但是 PPPoE 隧道有 8 个 bytes 的开销，所以 PPPoE 虚连接的 MTU 就是 ``1500-8=1492``，减掉 IPv4 包头（ 20 字节）和 TCP 包头（ 20 字节），可以得知 IPv4 下需要把 MSS 设为 1452 以下。IPv6 的包头是 40 字节，所以 IPv6 下需要把 MSS 设为 1432 以下。

因此如果你不使用 IPV6 , 可以将 mss 设置为 1452 .

```sh
set security flow tcp-mss all-tcp mss 1452
```

如果你使用 IPV6 ，请将 mss 设置为 1432 或者更低的值。

```sh
set security flow tcp-mss all-tcp mss 1432
```

原先的设置值是 1452 , 改成 1432 后，强制丢包现象消失，原先无法访问的某些站点就可以被访问了。

## IPV6 模式注意事项

如果在 PPPoE 上联连接成功后，发现 junos 没能获得 IPV6 地址，而且等待5分钟后依然没能获得 IPV6 地址，则需要进行以下步骤排查：

到 edit 模式下执行：

```sh
set protocols router-advertisement interface pp0.0
commit
```

到 cli 下执行：

```sh
% cli
> show ipv6 router-advertisement
```

如果什么也没有，稍等片刻再重新执行。

正常情况下会收到 RA, 需要看 M(Managed), O(Other configuration), A(Autonomous) 的值。

|      Method      |  M  |  O  |  A  |
| ---------------- | --- | --- | --- |
| SLAAC            | 0   | 0   | 1   |
| Stateless DHCPv6 | 0   | 1   | 1   |
| Statefull DHCPv6 | 1   | N/R | 0   |

假设遇到 M 为 1 的情况，请看下文 DHCPv6 PD . 
假设遇到 M 为 0 的情况，请看下文 RA Proxy .

### DHCPv6 PD

恭喜你，你很幸运，运营商会发 DHCPv6 PD 前缀给你，上面的配置直接就能使用，不需要额外设置。

### RA Proxy

首先，请确保系统更新到 ``JUNOS 22.1R1.10 built 2022-03-16`` 以及以上，因为旧版本不支持 RA proxy .

在 cli 下确认系统版本：

```sh
> show version 
```

然后开始配置：

```sh
# RA 上联端口
set protocols router-advertisement interface pp0.0 upstream-mode
set protocols router-advertisement interface pp0.0 downstream irb.100
set protocols router-advertisement interface pp0.0 solicit-router-advertisement-unicast
set protocols router-advertisement interface pp0.0 other-stateful-configuration
# RA 下联端口
set protocols router-advertisement interface irb.100 downstream-mode
# 打开 NDP 协议
set protocols neighbor-discovery
# PPPOE DHCPv6 ，如果你的junos不支持，请升级到前文说的新版本以及以上版本
del interfaces pp0 unit 0 family inet6
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-type autoconfig
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-ia-type ia-na
set interfaces pp0 unit 0 family inet6 dhcpv6-client client-identifier duid-type duid-ll
set interfaces pp0 unit 0 family inet6 dhcpv6-client req-option dns-server
set interfaces pp0 unit 0 family inet6 dhcpv6-client retransmission-attempt 3
# 设置 pp0.0 端口 NDP 和 DAD 代理
set interfaces pp0 unit 0 family inet6 ndp-proxy interface-unrestricted
set interfaces pp0 unit 0 family inet6 dad-proxy interface-unrestricted
# 设置 irb.100 端口 NDP 和 DAD 代理
set interfaces irb unit 100 family inet6 ndp-proxy interface-unrestricted
set interfaces irb unit 100 family inet6 dad-proxy interface-unrestricted
```

这样配置后，稍等一段时间，cli 下查看是否获得了 IPV6 地址：

```sh
> show ipv6 router-advertisement
> show dhcpv6 client binding detail
> show route table inet6.0
> show interfaces pp0.0 terse
> show system statistics icmp6
> show security flow session summary family inet6
> show security flow session family inet6
> show ipv6 neighbors
```


# 故障排错

## PPPoE

```
# 启用日志功能
set protocols ppp traceoptions level all
set protocols ppp traceoptions flag all
set protocols pppoe traceoptions file pppoe_log
set protocols pppoe traceoptions level all
set protocols pppoe traceoptions flag all
set interfaces pp0 traceoptions flag all

# 调试
request pppoe disconnect
request pppoe connect
show ppp interface pp0.0 extensive

# 查看日志
show log pppoe_log
show log pppd
```

# 参考资料

- How to configure SRX device as DDNS client and Destination NAT on Dynamic Untrust IP?
  <br><https://forums.juniper.net/t5/SRX-Services-Gateway/How-to-configure-SRX-device-as-DDNS-client-and-Destination-NAT/td-p/119650>
- NDP and DAD proxy support on multiple interfaces
  <br><https://www.juniper.net/documentation/us/en/software/junos/release-notes/22.1/junos-release-notes-22.1r1/topics/new-features/feature-descriptions/ipv6-11.html>
- Router Advertisement Proxy
  <br><https://www.juniper.net/documentation/us/en/software/junos/neighbor-discovery/topics/topic-map/newtopic-map.html>
