+++
title = "VMware ESXi 使用代理进行更新"
date = "2020-06-27"
tags = ["VMware"]
+++

默认情况下仅允许 `3128` 端口进行外部访问

如果需要使用非 `3128` 端口 HTTP 代理更新需要禁用防火墙

```plain
# disable firewall
esxcli network firewall unload

# enable firewall
esxcli network firewall load
```

## 参考资料

- <https://blog.ptsang.net/exsi6-install-patch-manually>
- <https://www.eknori.de/2019-09-01/esxi-6-7-update-no-space-left-on-device/>
- <https://pubs.vmware.com/vsphere-50/index.jsp?topic=%2Fcom.vmware.vcli.ref.doc_50%2Fesxcli_software.html>
- <https://pubs.vmware.com/vsphere-50/index.jsp?topic=%2Fcom.vmware.vsphere.security.doc_50%2FGUID-7A8BEFC8-BF86-49B5-AE2D-E400AAD81BA3.html>
