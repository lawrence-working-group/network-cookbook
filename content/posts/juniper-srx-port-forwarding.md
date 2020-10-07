+++
title = "Juniper SRX 端口转发"
date = "2020-10-07"
tags = ["JunOS"]
+++

```
# 设置目标地址和端口号
set security nat destination pool RDP-terminal-server address 192.168.1.2/32 port 3389

# 设置转发规则
set security nat destination rule-set default-inbound from interface pp0.0 # or routing-instance or zone
set security nat destination rule-set default-inbound rule 1 match destination-address 0.0.0.0/0
set security nat destination rule-set default-inbound rule 1 match destination-port 3389
set security nat destination rule-set default-inbound rule 1 then destination-nat pool RDP-terminal-server
```

注意 MX 系列那个 `service` 下的 NAT 规则配置能 commit，但是不会生效。

调试：

[Resolution Guide – SRX - Troubleshoot Destination NAT](https://kb.juniper.net/InfoCenter/index?page=content&id=KB21839&actp=METADATA)

上面几步做完以后还是不成功的话首先[启用 flow 日志](https://kb.juniper.net/InfoCenter/index?page=content&id=KB21757&actp=METADATA)，然后在日志里面找具体匹配了哪条。如果配置了 `persistent-nat` 的话，进来的数据包会优先查找 `persistent-nat` 表，可能会匹配上不正确的 flow，从日志中可以看到。这时候首先查看相应的 persistent NAT 规则和 flow：

```
show security nat source persistent-nat-table internal-ip 192.168.1.2 internal-port 3389
show security flow session destination-port 3389
```

然后把这些表项清理掉（可能需要多打几次）：

```
clear security flow session destination-port 3389
clear security nat source persistent-nat-table internal-ip 192.168.1.2 internal-port 3389
```
