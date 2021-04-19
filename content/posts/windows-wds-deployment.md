+++
title = "部署 Windows Deployment Services 用于 PXE 启动"
date = "2021-04-19"
tags = ["Windows", "PXE", "WDS", "Deployment"]
+++

今天打算给平板洗系统的时候发现找不到 U 盘，于是快进到家里部署 PXE。于是顺路装了个 Windows Deployment Services。
首先 Windows Deployment Services 现在已经不强制需要 AD 了，本文的部署指南也会基于 Domainless (Standalone) 模式
继续。注意在 Domainless 模式下，你的系统安装镜像也可以是基于 AD 认证的（TFTP Bootstrap 完成后，它本质就是个 SMB。）

## 服务部署

1. 安装 Windows Deployment Services Role，该 Role 的所有功能勾选。
2. 进入 Windows Deployment Services 管理界面，初始化服务器，选择 Standalone 模式。
3. 设置安装库目录。最好不要选系统盘。
4. 选择 Respond to all client computers。如果你有~~白名单~~许可名单的需求，你也可以选择 Respond only to known client computers 或者勾选下面的 Admin Approval。
5. 如果开放给所有机器安装（即不需要认证），可以考虑启用 SMB 匿名共享并检查 `REMINST` 共享的权限配置。

## 添加镜像

1. 准备一些 Windows ISO，把它们挂载上来。
2. 在 Install Images 下创建 Image Group，比如我家里有 `Win10_Client` 和 `WinBlue_Client`。右键 Image Group 添加 Installer Image。
3. 找到你的 ISO 下的 `install.wim` 并自动添加。如果有需要，可以重命名镜像（之后做也可以）。
4. 在 Boot Images 下右键并导入你的 `boot.wim`。
5. 重复 2-4 步骤，直到你导入了你需要的所有 SKU 和体系结构的镜像。

## 添加驱动（可选）

1. 在 Drivers 下创建你需要的驱动组，比如说 `SurfacePro3` / `SurfacePro4`。
2. 右键 Driver Node 选择 Add Driver Package，添加需要的驱动，然后选择之前创建的驱动组。
3. 可选：等待添加完成后，勾选修改 Filter，然后在弹出的窗口里添加 SMBIOS 匹配信息，使得特定组别的驱动只会在特定机器或系统版本上安装。

## 配置网络

1. 如果 WDS 服务器和部署机器在同个广播域下，理论上，只要 DHCP 服务器存活，理论上什么都不用做就能用了。
2. 如果 WDS 服务器和部署机器不在同个广播域下，需要配置 DHCP 服务器 / `ip helper` (Cisco 叫法）让客户端可以来 WDS 服务器获取一部分服务器信息，并且确保它们间网络互相可路由。
3. 如果觉得上述配置太烦，理论上可以在 DHCP 服务器上手动指派 Option 60/66/67，但是这样子就无法非常方便地实现不同体系结构 (e.g. ARM64 + x64) 和不同启动固件 (BIOS / UEFI) 自适应 PXE 启动了。如果不嫌麻烦，可以用 dnsmasq 来匹配体系结构和固件 (Thanks HarryChen)。

## TODO

本文还有一些内容没写，之后可能会更新：

- 用 WDS 启动 iPXE
- 自动 `unattend.xml`
- 根据 AD 安全组配置镜像可见性
- Multicast 传输装机器（节省批量装机的带宽消耗）
- Prestage 机器
