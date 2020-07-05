+++
title = "JunOS 更新"
date = "2020-05-29"
tags = ["JunOS"]
+++

# 准备工作

准备一个容量小于等于 4GB 的 U 盘，建立 MBR 分区表（一定要有），创建一个 FAT32 分区并格式化。把系统镜像直接装入分区根目录。如果找不到这样的 U 盘的话，准备一个 TFTP 服务器和充足的时间也可以。

如果目标系统存储空间不足，请事先[清理存储](https://www.juniper.net/documentation/en_US/junos/topics/task/troubleshooting/system-storage-cleanup-qfx-series.html)：

```
request system storage cleanup
```

# 更新系统

## 从正常工作的 JunOS 更新

### U 盘方式

先把文件复制到 `/var/tmp` 然后去 cli 里面启动更新。（如果不复制到 `/var/tmp`，视硬件和系统版本不同，会有各种不同报错；所以这里建议总是先复制到 `/var/tmp`。）

注意：SRX300/500 系列防火墙上请不要快速插拔 USB 设备，建议冷插拔。

```
root@% mount -t msdosfs /dev/da1s1 /mnt
root@% cp /mnt/junos-srxsme-18.2R3-S2.9.tgz /var/tmp
root@% umount /mnt
root@% cli
root> request system software add no-validate /var/tmp/junos-srxsme-18.2R3-S2.9.tgz
```

### TFTP 方式

JunOS 文档会告诉你说 `file copy` 命令支持 TFTP，但是其实它不支持。解决方法是去 BSD shell 里面调用 `tftp` 命令。

TFTP 传输时请稍安勿躁，传输系统镜像大约需要 30 分钟。

```
root@% cd /var/tmp
root@% tftp 192.168.1.100
tftp> get junos-srxsme-18.2R3-S2.9.tgz
tftp> quit
root@% cli
root> request system software add no-validate /var/tmp/junos-srxsme-18.2R3-S2.9.tgz
```

## 从 loader 安装

如果系统已经损坏，或者用正常方法无法安装，或者你想格式化硬盘从头再来，那么就需要从 loader 安装系统。这个方法会全盘格式化，清空所有文件和配置，请事先做好备份。

如果系统无法启动，那么开机后应该会直接看到 `loader>` 提示符。但是如果系统还能正常启动的话，就需要趁 `kernel...` 字样出现的时候狂按空格键来进入 loader。注意开机阶段有两次能按空格停下来的地方，如果你停下来以后没看到 `loader>` 的提示符，说明按早了。

### U 盘方式

```
install file:///junos-srxsme-18.4R3-S2.tgz
```

文件名是大小写不敏感的。`install` 命令的帮助会告诉你应该使用 `usb://` 开头的 URL；可惜这也是假的，你得使用 `file://`。

### TFTP 方式

我没试过，看[文档](https://www.juniper.net/documentation/en_US/junos/topics/topic-map/install-software-on-srx.html#id-installing-junos-os-on-srx-series-devices-from-the-boot-loader-using-a-tftp-server)吧。

# 复制系统

## 和备份分区同步

```
request system snapshot slice alternate
```

## 把系统复制到 U 盘

注意，这会格式化你的 U 盘。

```
request system snapshot
```

# 常见问题排错

## 第一次启动以后无法 commit 配置

现象：commit 配置时报错：

```
[edit security utm utm-policy junos-av-policy anti-virus http-profile]
'http-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-policy anti-virus ftp upload-profile]
'upload-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-policy anti-virus ftp download-profile]
'download-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-policy anti-virus smtp-profile]
'smtp-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-policy anti-virus pop3-profile]
'pop3-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-policy anti-virus imap-profile]
'imap-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus http-profile]
'http-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus ftp upload-profile]
'upload-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus ftp download-profile]
'download-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus smtp-profile]
'smtp-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus pop3-profile]
'pop3-profile junos-av-defaults'
An anti-virus profile must be defined
[edit security utm utm-policy junos-av-wf-policy anti-virus imap-profile]
'imap-profile junos-av-defaults'
An anti-virus profile must be defined
error: commit failed: (statements constraint check failed)
```

原因：你在不被支持的老设备上启动了新版本系统。老设备的默认配置没有更新，commit 时合并配置产生了一些无法被新版系统识别的配置。

解决方案：`set security utm apply-groups-except junos-defaults` 阻止默认配置被合并即可。
