+++
title = "Hyper-V 直通 PCIe 设备（如网卡）到虚拟机"
date = "2020-06-04"
tags = ["windows", "hyper-v", "Cisco", "pcie"]
+++

**编者按：SR-IOV 和 PCIe 设备直通属于非常新的技术，且大部分民用设备有非常严重的兼容性问题。在使用前，请务必和你的平台与设备厂商确认兼容性。使用不兼容的设备可能导致虚拟机或物理机失去响应，设备进入未知的状态，电脑起火等诸多后果。编者不对任何尝试提供担保。**

## 先决条件

- Hyper-V 安装并启用（废话）
- 处理器支持 IOMMU (Intel VT-d / AMD-Vi / ARM SMMU) 并启用
- 平台固件启用了 PCIe ACS (Access Control Services)
- 一块不使用 Line Interrupt 的 PCIe 设备，最好能支持 PCIe ACS (Access Control Services)

## 已知问题

- 直通设备的虚拟机不能被暂停或热迁移，所以自动关机动作必须为 ACPI shutdown 或者硬关机。
- 如果是 Linux 虚拟机，需要 Hyper-V PCI Bus 驱动 `hv_pci`。
- 只有 Windows Server Hyper-V 支持设备直通。

## 测试过的设备

- HPE ProLiant ML30 Gen 10 + Mellanox ConnectX-4 Lx：原生支持 IOMMU 和 ACS，不需要 `-Force`。
- Dell R910 + Intel X520-DA2：原生支持 IOMMU；ACS 支持不全，需要加 `-Force`。

## 检查设备信息

从 [这里](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/hyperv-tools/DiscreteDeviceAssignment/SurveyDDA.ps1) 下载检查脚本并运行。如果需要直通的设备所有报告全绿 （i.e. 不报错），那么复制 PCIe 设备路径备用。它应该长得像 `PCIROOT(0)#PCI(1C00)#PCI(0000)` 这样。

如果这个脚本告诉你设备存在 Line Interrupt：

```
All of the interrupts are line-based, no assignment can work.
```

那么这个设备不受支持。

如果这个脚本告诉你 BIOS 需要设备不离开它控制的内存：

```
BIOS requires that this device remain attached to BIOS-owned memory.  Not assignable.
```

请跟 OEM 开票以获取正确支持 DDA 的 UEFI/BIOS 固件。

如果这个脚本告诉你其他设备可能可以访问这个设备：

```
Traffic from this device may be redirected to other devices in the system.  Not assignable.
```

你依然有可能直通网卡（但是不安全）。在确定没有敏感工作负载的情况下，修改脚本如下：

```powershell
$locationpath = ($pcidev | get-pnpdeviceproperty DEVPKEY_Device_LocationPaths).data[0]
$acsUp =  ($pcidev | Get-PnpDeviceProperty $devpkey_PciDevice_AcsCompatibleUpHierarchy).Data
    if ($acsUp -eq $devprop_PciDevice_AcsCompatibleUpHierarchy_NotSupported) {
        write-host -ForegroundColor Red -BackgroundColor Black "Traffic from this device may be redirected to other devices in the system.  Not assignable."
        if ($null -ne $locationpath) {
            $locationpath
        }
        continue
    }
```

其他设备（例如传统 PCI 设备）也不受支持。

## 准备设备和虚拟机

在设备管理器里禁用准备直通的设备。打开 PowerShell，赋变量 `$locationpath` 为刚刚获得的 PCIe 设备路径。然后运行：

```powershell
# 如果设备提供了直通用迁移驱动，那么不需要加 -Force
# 有原生 ACS 支持的也不需要
Dismount-VMHostAssignableDevice -LocationPath $locationpath -Force
```

如果这一步还是失败了，那么你的平台或者设备可能不受支持。如果这一步成功了，请往下继续看。

赋变量 `$VmName` 为虚拟机名，然后运行：

```powershell
# 直通设备的虚拟机不能被暂停或热迁移，所以自动关机动作必须为 ACPI shutdown 或者硬关机
Set-VM -Name $VmName -AutomaticStopAction TurnOff

# 设置虚拟机缓存类型和 MMIO 区域。不同的设备对 MMIO 要求可能不同，具体可以参考
# https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/plan/plan-for-deploying-devices-using-discrete-device-assignment
Set-VM -VMName $VmName -GuestControlledCacheTypes $true  -LowMemoryMappedIoSpace 3Gb -HighMemoryMappedIoSpace 33280Mb

# 将设备通入虚拟机
Add-VMAssignableDevice -VMName $VmName -LocationPath $locationpath
```

## 将设备从虚拟机里移除

关闭虚拟机。赋变量 `$locationpath` 为刚刚获得的 PCIe 设备路径，赋变量 `$VmName` 为虚拟机名，然后运行：

```powershell
# 将设备从虚拟机里移除
Remove-VMAssignableDevice -VMName $VmName -LocationPath $locationpath
# 挂载设备回 Management OS
Mount-VMHostAssignableDevice -LocationPath $locationpath
```

然后在设备管理器里重新启用设备。

## 例子: CSR1000v 16.12

```
UplinkRouter#sh platform software vnic-if database
vNIC Database
  eth00_159127288xxxxxxxxxxx
    Device Name : Gi1
    Driver Name : ixgbe
    MAC Address : xxxx.xxxx.xxxx
    PCI DBDF    : b605:00:00.0
    UIO device  : yes
    Management  : no
    Status      : supported
```

## 注意事项

- 同一个设备只需要跑一次 `Add-VMAssignableDevice`。这个命令有概率不检查去重，会导致你的虚拟机无法启动。如果遇到这种情况，那么先把设备从虚拟机里移除，然后重新添加一次。
