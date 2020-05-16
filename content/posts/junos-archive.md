+++
title = "JUNOS Archive - GitHub Bridge"
date = "2020-05-16"
tags = ["Juniper", "JUNOS"]
+++

## JUNOS Archive - GitHub Bridge

由于 JUNOS 是一个有状态的系统，意外掉电会直接炸掉 primary 分区
掉进 backup 分区里面启动（老配置），但是你又回不去

所以我就做了一个 <https://github.com/NiceLabs/juniper-archive-agent>
利用 <https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/junos-software-system-management-router-configuration-archiving.html> 提交时自动上传 FTP 的特性
做了一个虚拟的 FTP 服务器用来接收 JUNOS 上传进来的文件，且同步储存到 GitHub 上

这样在丢失备份时可以直接恢复以前上传的副本
