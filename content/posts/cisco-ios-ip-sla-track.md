+++
title = "Cisco IOS 使用 IP SLA 状态作为静态路由的开关"
date = "2020-11-15"
tags = ["Cisco", "IOS", "SLA"]
+++

Cisco IOS 和 IOS XE 作为最普及的路由器和交换机操作系统，内建强大的状态跟踪联动能力。

我们配置网络设备时，希望设备能主动发现网络故障，及时作出相应调整，并将故障状态传递给网络的其他部分，减少故障恢复时的人工干预。

## 本文涉及 Cisco IOS 中的三个功能
1. 静态路由随 track 状态开关
1. 使用 IP SLA 检查指定 IP 是否响应 ICMP ECHO REQUEST（即 ping）
1. Enhanced Object Tracking（即前述 track）聚合多个 IP SLA 检查的结果

## 案例介绍
### 配置要求
给出一种判断互联网接入链路是否可用的方法，如果此互联网接入链路不可用，从本机路由表中删除对应的默认路由，使得依赖此默认路由的动态路由协议向其邻居撤回默认路由。

### 基本思路
1. 持续跟踪互联网上两个 IP 地址的 ICMP 可达性
1. 如果它们中至少有一个可达，启用默认路由，同时禁用为这两个地址添加的静态主机路由。
1. 如果两个均不可达，禁用/撤回默认路由。

### 相关配置
本文使用 192.0.2.1, 198.51.100.1 和 203.0.113.1 作为示例，这三个地址是 RFC 5735 中规定的文档专用地址，不在互联网上使用，读者应该自己选择合适的地址，与地址使用方联系获得许可，避免被误判为拒绝服务攻击流量。

- 配置两个 IP SLA 测试
  1. 发送的 ICMP 数据报文结构 C15C0105 是 CISCO IOS 的另类拼写，可自定义。
  1. ISP-A 是使用的本机配置的对应此运营商的 VRF 名称，如果在全局路由表使用，可以不写这一行
  1. tag 是对此 IP SLA 测试的别名，用于帮助识别
  1. threshold 40 定义了 40ms 以上的 RTT 时间为超出上限，timeout 80 定义了 80ms 以上的 RTT 时间判为超时错误。
  1. 最后一行 ip sla group schedule 将两个 IP SLA 测试统一启动，每一个周期都是 10 秒，启动间隔 5s，立即启动，永远执行。

  ```
  ip sla 1
   icmp-echo 198.51.100.1 source-interface GigabitEthernet1
   request-data-size 1200
   tos 184
   data-pattern C15C0105
   vrf ISP-A
   tag ping-TESTNET-2
   threshold 40
   timeout 80
   frequency 10
   history distributions-of-statistics-kept 5
   history lives-kept 2
   history filter all
  !
  ip sla 2
   icmp-echo 203.0.113.1 source-interface GigabitEthernet1
   request-data-size 1200
   tos 184
   data-pattern C15C0105
   vrf ISP-A
   tag ping-TESTNET-3
   threshold 100
   timeout 200
   frequency 10
   history distributions-of-statistics-kept 5
   history lives-kept 2
   history filter all
  !
  ip sla group schedule 1 1-2 schedule-period 10 start-time now life forever
  ```

- 配置调用上述 IP SLA 测试结果的 Tracking
  1. track 1 状态与 IP SLA 测试 1 联动
  1. track 2 状态与 IP SLA 测试 2 联动
  1. 在跟踪的 IP SLA 变为 DOWN 之后 31 秒内不改变 track 自身状态，如果这段等待期内 IP SLA 变为 UP，track 状态不会震荡。同理，若 track 从 DOWN 状态恢复，需要等待 61 秒内没有 IP SLA DOWN 的事件。
  1. track 3 状态取决于前两个跟踪目标（可以配置更多），如果小于 24% 则判为无效，如果大于 49% 则判为有效。在只配置了两个 object 的情况下，一个 object 状态为 UP，则贡献 50%，整体判为有效。
  1. track 10 的状态与 ISP-A 这个 VRF 的路由表中是否存在默认路由联动
  1. track 11 的状态与 track 10 状态相反
  ```
  track 1 ip sla 1
   delay down 31 up 61
  !
  track 2 ip sla 2
   delay down 31 up 61
  !
  track 3 list threshold percentage
   object 1
   object 2
   threshold percentage down 24 up 49
  !
  track 10 ip route 0.0.0.0 0.0.0.0 reachability
   ip vrf ISP-A
  !
  track 11 list boolean and
   object 10 not
  ```

- 配置与上述 track 联动的静态路由
  1. 两个被测试地址的主机路由与默认路由的相反状态关联，即存在默认路由时，这两条主机路由不会被安装。
  1. 默认路由与链路可用性关联，即可以到达两个被测试地址时，安装默认路由。
  1. 以下配置不会形成震荡的原因是，两个被测试地址可以通过默认路由离开本路由器。
  ```
  ip route vrf ISP-A 198.51.100.1 255.255.255.255 192.0.2.1 name TESTNET-1 track 11
  ip route vrf ISP-A 203.0.113.1 255.255.255.255 192.0.2.1 name TESTNET-2 track 11
  ip route vrf ISP-A 0.0.0.0 0.0.0.0 192.0.2.1 name ISP-A-DEFAULT track 3
  ```

## 参考资料
IP SLA 和 Tracking 的选项及其丰富，读者可参考
1. [IP SLAs Configuration Guide, Cisco IOS XE 16](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipsla/configuration/xe-16/sla-xe-16-book.html)
1. [IP Application Services Configuration Guide, Cisco IOS XE 16 - Configuring Enhanced Object Tracking](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/configuration/xe-16/iap-xe-16-book/iap-eot.html)
