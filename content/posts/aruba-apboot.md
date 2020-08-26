+++
title = "Aruba APBoot 模式下的升级"
date = "2020-08-24"
tags = ["HPE", "Aruba"]
+++

## 进入 APBoot 模式

接上 Console

```
APBoot 1.2.8.1 (build 34939)
Built: 2012-08-17 at 12:54:18

Model: AP-13x
CPU:   88F6560 A0 (DDR3)
Clock: CPU 1600MHz, L2 533MHz, SysClock 533MHz, TClock 200MHz
DRAM:  256MB
POST1: passed
Flash: 16 MB
Power: 802.3at POE
LAN:   done
PHY:   done
PEX 0: RC, link up, x1
       bus.dev fn venID devID class  rev    MBAR0    MBAR1    MBAR2    MBAR3
       00.00   00  11ab  6560 00005   02 f1000000 00000000 00000000 00000000
       00.01   00  168c  0030 00002   01 90000000 00000000 00000000 00000000
PEX 1: RC, link up, x1
       bus.dev fn venID devID class  rev    MBAR0    MBAR1    MBAR2    MBAR3
       01.00   00  11ab  6500 00005   02 f1000000 00000000 00000000 00000000
       01.01   00  168c  0030 00002   01 94000000 00000000 00000000 00000000
Net:   eth0, eth1
Radio: ar9390#0, ar9390#1

Hit <Enter> to stop autoboot:  0
apboot> ?
?              - alias for 'help'
boot           - boot the OS image
clear          - clear the OS image or other information
dhcp           - invoke DHCP client to obtain IP/boot params
factory_reset  - reset to factory defaults
help           - print online help
mfginfo        - show manufacturing info
osinfo         - show the OS image version(s)
ping           - send ICMP ECHO_REQUEST to network host
printenv       - print environment variables
purgeenv       - restore default environment variables
reset          - Perform RESET of the CPU
saveenv        - save environment variables to persistent storage
setenv         - set environment variables
tftpboot       - boot image via network using TFTP protocol
upgrade        - upgrade the APBoot or OS image
version        - display version
```

请在 2 sec 以内按下 <kbd>Enter</kbd> 进入 APBoot 模式

## 升级固件

```
# 设置静态地址（如果 DHCP 可用可以不用设置）
apboot> setenv ipaddr 192.168.1.100
apboot> setenv netmask 255.255.255.0
apboot> setenv gatewayip 192.168.1.1

# 设置 TFTP 服务器地址
apboot> setenv serverip 192.168.1.101

# 清除 OS 镜像（可选）
apboot> clear os 0

# 升级 OS 镜像
apboot> upgrade os [file-name]

# 恢复默认配置信息
apboot> factory_reset

# 重启
apboot> reset
```

## 备忘录

1. 默认密码

   默认账号均为 `admin`

   以 ArubaOS 8 作为分界线，以前默认密码为 `admin`，以后默认密码为本机序列号

   本机序列号在在自签的证书中也有包含

2. 某位不透露姓名的 Aruba Staff 表示，建议使用 `printenv` 事先备份正常状态下的 `env` 信息。

   原话如下：

   ```
   ***, [20.08.20 00:12]
   别抹了 apboot 里的 env 就行

   ***, [20.08.20 00:12]
   那玩意弄起来挺麻烦
   ```

3. UART 线序定义

   如果你拿到的型号是 `103` / `305` 等型号的 Aruba AP，那么的 Console 一组 UART PINOUT

   按照 Console 文字方向的线序依次为：

   ```
   PIN 1 - GND
   PIN 2 - TX
   PIN 3 - RX
   PIN 4 - 3.3V
   ```

   see <https://support.arubanetworks.com/Documentation/tabid/77/DMXModule/512/EntryId/25044/Default.aspx>

   see <https://community.arubanetworks.com/t5/Wireless-Access/How-I-can-console-to-AP-315/td-p/312825>
