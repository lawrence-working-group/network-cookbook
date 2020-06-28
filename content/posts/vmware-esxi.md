+++
title = "VMware ESXi 笔记"
date = "2020-06-27"
tags = ["VMware"]
+++

## 使用代理

默认情况下仅允许 `3128` 端口进行外部访问

如果需要使用非 `3128` 端口 HTTP 代理更新需要禁用防火墙

```
# disable firewall
esxcli network firewall unload

# enable firewall
esxcli network firewall load
```

## 补丁包

<https://my.vmware.com/group/vmware/patch>

```
# update from depot
esxcli software vib update -d /vmfs/volumes/datastore/update-from-esxi6.7-6.7_update02.zip

# reboot device
reboot
```

## 参考资料

- <https://blog.ptsang.net/exsi6-install-patch-manually>
- <https://www.eknori.de/2019-09-01/esxi-6-7-update-no-space-left-on-device/>
- <https://blog.open4j.com/2019/06/02/update-esxi/>
- <https://pubs.vmware.com/vsphere-50/index.jsp?topic=%2Fcom.vmware.vcli.ref.doc_50%2Fesxcli_software.html>
- <https://pubs.vmware.com/vsphere-50/index.jsp?topic=%2Fcom.vmware.vsphere.security.doc_50%2FGUID-7A8BEFC8-BF86-49B5-AE2D-E400AAD81BA3.html>
