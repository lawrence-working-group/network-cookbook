+++
title = "思科设备的主机名和域名设置"
date = "2019-12-22"
tags = ["思科"]
+++

## Cisco IOS，IOS XE

```console
hostname <主机名>
ip domain-name <域名>
```

注意：在 IOS 设备里，SSH 服务器依赖 RSA 密钥对的生成，而 RSA 密钥对的生成，依赖本机的主机名和域名的存在。

## Cisco ASA

```console
hostname <主机名>
domain-name <域名>
```

与 ASA 里其他的命令类似，对应 IOS 中命令开头的 `ip` 大部分情况下是不需要的，相似的还有 `route` 命令配置静态路由，而非 `ip route`。
