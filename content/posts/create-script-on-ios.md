+++
title = "在思科 IOS 设备上创建命令脚本"
date = "2019-12-30"
tags = ["Cisco", "TCL"]
+++

由于经典 IOS 没有提供类似 JunOS 的 commit 机制，任何配置指令都会立即生效，这使得调整接口 VRF、设置路由协议 `passive-interface default` 等配置可能产生破坏性影响，例如中断当前的终端会话等。

结合本文创建脚本的步骤和《远程配置时防止把自己锁在外面》的方法，可以较好的规避配置时彻底断线的风险。

以调整 Gi0 接口的 VRF 为例，如果进入接口配置模式直接输入 `vrf forwarding TESTNET`，则接口上的 IP 和 IPv6 地址配置会被删除，假如当前的 SSH 会话是通过 Gi0 接口到达设备的，则 SSH 会话会立即中断，且无法恢复。使用以下方法，在断线后接口的 IP 地址配置恢复，可以重新发起 SSH 会话。

现在的配置：

```console
interface GigibitEthernet0
 ip address 192.168.1.1 255.255.255.0
 ipv6 address 2001:db8::1/64
end
```

目标配置：

```console
interface GigibitEthernet0
 vrf forwarding TESTNET
 ip address 192.168.1.1 255.255.255.0
 ipv6 address 2001:db8::1/64
end
```

使用 Cisco IOS 内置的 TCL 环境，在文件系统内写脚本文件。

```console
tclsh
puts [ open "flash:CHANGEVRF.CFG" w+ ] {
interface Gi0
 vrf forwarding TESTNET
 ip address 192.168.1.1 255.255.255.0
 ipv6 address 2001:db8::1/64
exit
end
}
tclquit
```

接下来执行上述配置脚本（与 running-config 合并）

```console
copy flash:CHANGEVRF.CFG running-config
```

SSH 会话可能会中断，重新登陆到设备，验证配置结果，并保存当前配置。
