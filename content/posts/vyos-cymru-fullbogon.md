+++
title = "为 VyOS 配置 Cymru BGP Fullbogon 会话"
date = "2020-02-25"
tags = ["VyOS", "BGP"]
+++

[Cymru](https://www.team-cymru.com/bogon-reference-bgp.html) 通过 BGP 分发动态的 Bogon IP 列表。其官网提供了许多[示例配置](https://www.team-cymru.com/bgp-examples.html)。
本文基本上只是把示例配置翻译到 VyOS 语法。

```
set policy community-list cymru rule 1 regex 65332:888
set policy community-list cymru rule 1 action permit

set policy prefix-list deny-all rule 10 action 'deny'
set policy prefix-list deny-all rule 10 le '32'
set policy prefix-list deny-all rule 10 prefix '0.0.0.0/0'

set policy prefix-list6 deny-all rule 10 action 'deny'
set policy prefix-list6 deny-all rule 10 le '128'
set policy prefix-list6 deny-all rule 10 prefix '::/0'

set policy route-map cymru-bogons-v4 rule 1 action 'permit'
set policy route-map cymru-bogons-v4 rule 1 match community community-list 'cymru'
set policy route-map cymru-bogons-v4 rule 1 set ip-next-hop '192.0.2.1'

set policy route-map cymru-bogons-v6 rule 1 action 'permit'
set policy route-map cymru-bogons-v6 rule 1 match community community-list 'cymru'
set policy route-map cymru-bogons-v6 rule 1 set ipv6-next-hop global 2001:DB8:0:DEAD:BEEF::1

set protocols static route 192.0.2.1/32 blackhole
set protocols static route6 2001:DB8:0:DEAD:BEEF::1/128 blackhole

set protocols bgp <Your ASN> neighbor A.B.C.D address-family ipv4-unicast prefix-list export 'deny-all'
set protocols bgp <Your ASN> neighbor A.B.C.D address-family ipv4-unicast route-map import 'cymru-bogons-v4'
set protocols bgp <Your ASN> neighbor A.B.C.D address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp <Your ASN> neighbor A.B.C.D ebgp-multihop '255'
set protocols bgp <Your ASN> neighbor A.B.C.D password <BGP Session Password>
set protocols bgp <Your ASN> neighbor A.B.C.D remote-as '65332'
set protocols bgp <Your ASN> neighbor A.B.C.D update-source <Your IPv4 Address>

set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX address-family ipv6-unicast prefix-list export 'cymru-bogons-v6'
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX address-family ipv6-unicast route-map import 'deny-all'
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX address-family ipv6-unicast soft-reconfiguration inbound
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX ebgp-multihop '255'
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX password <BGP Session Password>
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX remote-as '65332'
set protocols bgp <Your ASN> neighbor XXXX:XXX:XXXX::XXXX:XXXX update-source <Your IPv6 Address>

set protocols bgp <Your ASN> parameters graceful-restart
```
