+++
title = "AAA 新模型配置"
date = "2019-12-22"
tags = ["思科 Cisco", "AAA"]
+++

## Cisco IOS/IOS XE

在启用 `aaa new-model` 之前，务必小心，防止把远程操作的自己锁在外面。参考 avoid-lockout 一文进行操作。
在启用之前，还需要配置一个或多个本地用户，即使要使用 TACACS+ 或是 RADIUS 认证管理用户，也应该有本地用户作为 fallback 方案，请参考本地用户管理。

完成上述两步后，在配置模式输入以下命令：

```console
aaa new-model
aaa authentication login default local
aaa authorization console
aaa authorization exec default local if-authenticated
aaa authorization commands 15 default local if-authenticated
```

由于授权方式的变更，当前用户可能没有保存配置的权限，建议使用刚才创建的高权限本地用户登陆设备，保存配置，取消计划中的 reload。
