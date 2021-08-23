+++
title = "Juniper SRX 端口转发"
date = "2020-10-07"
tags = ["JunOS"]
+++

```
# 如果开启了 persistent-nat，建议避开需要端口转发的端口
set security nat source pool-default-port-range 4096 to 63487

# 设置目标地址和端口号
set security nat destination pool RDP-terminal-server address 192.168.1.2/32 port 3389

# 设置转发规则
set security nat destination rule-set default-inbound from interface pp0.0 # or routing-instance or zone
set security nat destination rule-set default-inbound rule 1 match destination-address 0.0.0.0/0
set security nat destination rule-set default-inbound rule 1 match destination-port 3389
set security nat destination rule-set default-inbound rule 1 then destination-nat pool RDP-terminal-server

TL;DR，若要转发多个端口（一个范围），可参考以下配置

# 设置转发的目标地址
set security nat destination pool RDP-terminal-server 192.168.1.2/32

# 设置转发端口范围及规则
set security nat destination rule-set default-inbound rule port-forward-range match destination-address <Your Public IP Address>
set security nat destination rule-set default-inbound rule port-forward-range match destination-port 10000 to 20000
set security nat destination rule-set default-inbound rule port-forward-range then destination-nat pool RDP-terminal-server
```

注意事项：
* 在 SRX 上，MX 系列那个 `service` 下的 NAT 规则配置能 commit，但是不会生效
* Junos 收到包后会先匹配 flow 并直接应用 flow 上引用的 NAT 规则，如果匹配失败再做 NAT
* Junos 在默认设置下，返回包的路由是在 flow 第一个包所在的 routing instance 查询的

调试：

首先看是否被 firewall filter 拒绝掉或者 security policy 拒绝掉，前者在相应 filter 里面加 count 语句来看，后者可以用 `show security match-policies` 测试。

然后进行 flow 建立过程调试和 NAT 过程调试：

* [SRX Getting Started -- Troubleshooting Traffic Flows and Session Establishment](https://kb.juniper.net/InfoCenter/index?page=content&id=kb16110)
* [Resolution Guide – SRX - Troubleshoot Destination NAT](https://kb.juniper.net/InfoCenter/index?page=content&id=KB21839&actp=METADATA)

还是不成功的话再[启用 flow 日志](https://kb.juniper.net/InfoCenter/index?page=content&id=KB21757&actp=METADATA)，在日志里面找具体匹配了什么规则导致 flow 建立/匹配失败。如果[配置了 `persistent-nat` 的话](https://www.blackhole-networks.com/SRXNAT/snat_persist.html)，进来的数据包会优先查找 `persistent-nat` 表，可能会匹配上不正确的 flow，从日志中可以看到。这时候首先查看相应的 persistent NAT 规则和 flow：

```
show security nat source persistent-nat-table internal-ip 192.168.1.2 internal-port 3389
show security flow session destination-port 3389
```

然后把这些表项清理掉（可能需要多打几次）：

```
clear security flow session destination-port 3389
clear security nat source persistent-nat-table internal-ip 192.168.1.2 internal-port 3389
```
