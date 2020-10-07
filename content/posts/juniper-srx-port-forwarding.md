+++
title = "Juniper SRX 端口转发"
date = "2020-10-07"
tags = ["JunOS"]
+++

```
# 设置目标地址
set security nat destination pool RDP-terminal-server address 192.168.1.2/32

# 设置转发规则
set security nat destination rule-set default-inbound from interface pp0.0 # or routing-instance or zone
set security nat destination rule-set default-inbound rule 1 match destination-address 0.0.0.0/0
set security nat destination rule-set default-inbound rule 1 match destination-port 3389
set security nat destination rule-set default-inbound rule 1 then destination-nat pool RDP-terminal-server
```

注意 MX 系列那个 `service` 下的 NAT 规则配置能 commit，但是不会生效。
