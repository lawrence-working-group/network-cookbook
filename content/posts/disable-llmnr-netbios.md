+++
title = "在 Windows 和 Linux 系统上禁用 LLMNR 以及 NetBIOS"
date = "2019-12-30"
tags = ["安全","Windows"]
+++

## 什么是 LLMNR

LLMNR （Link-Local Multicast Name Resolution、链路多播本地名称解析、mDNS）是一个基于 DNS 格式的名称解析协议。支持 LLMNR 的主机可以在 DNS 服务不可用的时候在本地网络多播 LLMNR 请求从其它支持 LLMNR 的主机解析域名。此协议同样可以让主机解析本地主机域名对应的 IP。

## 什么是 NetBIOS

NetBIOS（网络基本输入输出系统）是一种目前大多基于 TCP/IP 协议的会话层协议/API。它能够提供名称登录，名称解析等服务。NetBIOS 在 Windows 安装 TCP/IP 栈的时候就会被携带安装，并且默认打开。

## 为什么要禁用 LLMNR 和 NetBIOS

LLMNR 客户端（请求者）会认为应答是权威的，所以当有客户尝试解析域名的时候，恶意节点可以返回错误信息以至于客户端认为恶意节点的 IP 是域名对应的 IP，或者解析出不存在的 IP。由此攻击又可展开 MITM 以及 replay 等攻击。

具体的攻击实例可以参考文章[《NetBIOS 名称欺骗和 LLMNR 欺骗》](https://www.jianshu.com/p/a22dd51ce27a)。该文章展示了使用 Kali 工具 [Reponder](https://github.com/SpiderLabs/Responder) 进行 LLMNR 和 NetBIOS 欺骗。

## LLMNR

### Windows 禁用方法

使用本地组策略编辑器（gpedit.msc）依次打开如下项目：

> 计算机配置 -> 管理模板 -> 网络 -> DNS 客户端 -> 关闭多播名称解析

在英文系统中，该项名称为：

> Computer Configuration -> Administrative Templates -> Network -> DNS Client -> Turn Off Multicast Name Resolution

接下来，只需要将该 GPO 项目设置为 “禁用” 即可。

### Linux (Debian/Ubuntu) 禁用方法

在 `/etc/systemd/resolved.conf` 文件中，修改或新创建：

```conf
LLMNR=no
```

## NetBIOS

### Windows 禁用方法

在控制面板中依次打开：

> 网络和 Internet -> 网络连接 -> 右键想要更改的网卡 -> 选择 “属性” -> 点击 “Internet 协议版本 4 (TCP/IPv4)” -> 点击属性属性 -> 点击高级

### Linux 禁用方法

Linux 系统下, [Samba](https://zh.wikipedia.org/wiki/Samba) （提供了许多 Windows 相关的服务实现）中的 [`nmbd` 服务](https://www.samba.org/samba/docs/current/man-html/nmbd.8.html)会响应 NetBIOS 包。如果需要关闭 NetBIOS，可以编辑 `smb.conf` 文件。该文件在基于 Debian 的系统上位置为 `/etc/samba/smb.conf`。在该文件中，添加或修改如下行：

```conf
disable netbios = yes
```

`/etc/init/nmbd.conf` 会检查该项是否为 yes，如果该项为 yes，则 nmbd 服务进程不会启动。
