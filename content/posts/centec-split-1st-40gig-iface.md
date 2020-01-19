+++
title = "盛科交换机将第一个 40Gbps 接口分成四个（板载） 10Gbps 接口"
date = "2020-01-19"
tags = ["盛科", "Centec", "交换机"]
+++

## 适用于盛科 E580-20Q4Z 交换机
此交换机默认配置情况下只有 20 个 40Gbps QSFP+ 接口，中间夹着 4 个 100Gbps 接口可降速为 40Gbps 接口来使用。

在这些接口之前另有 4 个一起的 SFP+ 接口，内部连接到第一个 40Gbps 接口 eth-0-1。

使用以下命令可以启用这四个 SFP+ 10Gbps 接口。保存配置后需重启。

进入配置模式：

```
split interface eth-0-1 10giga
switch interface eth-0-1 sfp
exit
write
reload
y
```