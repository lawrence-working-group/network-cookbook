+++
title = "Juniper SRX 家用路由器配置"
date = "2020-05-29"
tags = ["JunOS"]
+++

# 恢复出厂设置

如果登录不了，可能需要[重置 root 密码](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/recovering-root-password.html) 或者从 loader 安装系统以格式化硬盘。

进入系统以后，使用

```shell
request system zeroize
```
 
恢复出厂设置。
 
# 设置 root 密码
 
设备启动以后，使用默认用户名 `root` 空密码登录，然后：
 
```shell
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

# Port-restricted cone NAT
set security nat source interface port-overloading off
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat permit any-remote-host
set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface persistent-nat inactivity-timeout 600
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
set interfaces pp0 unit 0 family inet mtu 1492
set interfaces pp0 unit 0 family inet negotiate-address
set routing-options static route 0.0.0.0/0 next-hop pp0.0
set security flow tcp-mss all-tcp mss 1452
set security zones security-zone untrust interfaces ge-0/0/0.0
set security zones security-zone untrust interfaces pp0.0

# IPv6
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
