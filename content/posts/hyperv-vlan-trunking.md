+++
title = "Hyper-V 建立 VLAN Trunk 接口"
date = "2020-06-11"
tags = ["Windows", "Hyper-V"]
+++

众所周知，如果你在 Hyper-V GUI 里面去掉 VLAN tagging 选项，那么你会获得一个 untagged access 接口，也就是说所有带着 VLAN tag 的帧就进不来了。但是其实 Hyper-V 是支持把一个接口作为真正的 VLAN trunk 接口的。

首先我们需要获得接口的名字：

```powershell
PS C:\Windows\system32> Get-VMNetworkAdapterVlan -VmName "Linux"

VMName   VMNetworkAdapterName Mode     VlanList
------   -------------------- ----     --------
Linux    Network Adapter      Untagged
```

然后设置这个接口为 trunk 接口：

```powershell
PS C:\Windows\system32> Set-VmNetworkAdapterVlan -VmName "Linux" -VmNetworkAdapterName "Network Adapter" -Trunk -NativeVlanId 0 -AllowedVlanIdList "1-4094"
```

最后验证配置：

```powershell
PS C:\Windows\system32> Get-VMNetworkAdapterVlan -VmName "Linux"

VMName   VMNetworkAdapterName Mode  VlanList
------   -------------------- ----  --------
Linux    Network Adapter      Trunk 0,1-4094
```

注：Cisco CSR1000v 和这个功能一起用会有一些鬼故事，不建议这么配置。
