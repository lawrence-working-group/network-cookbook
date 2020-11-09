+++
title = "Cisco CSR1000V 虚拟路由器载入初始配置"
date = "2020-11-09"
tags = ["Cisco", "CSR1000V"]
+++

从 QCOW2 镜像启动的 CSR1000V 会自行生成一个并不好用的默认配置，通过光盘镜像提供 iosxe_config.txt 文件，可以将其中配置应用到新创建的 CSR1000V 虚拟路由器中。

## 步骤

1. 创建 iosxe_config.txt 文件，可使用 IOS XE configure 模式下的语法。

2. 创建一个仅包含上述文件的 ISO 光盘镜像。Linux 环境下使用如下命令：
```
mkisofs -o csr1000v-initcfg.iso -l --iso-level 2 iosxe_config.txt
```

3. 将生成的光盘镜像以 IDE 接口挂载到虚拟机。

4. 启动虚拟机，验证配置，查看 bootflash:/cvac.log，包含上述配置应用到系统过程中的详细日志。
