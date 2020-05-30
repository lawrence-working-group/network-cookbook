+++
title = "清理网络设备命令历史"
date = "2020-05-22"
tags = ["RouterOS", "Mikrotik", "思科 Cisco", "IOS", "命令历史"]
+++

## RouterOS
通过 SSH 或 Webfig 的 Terminal，执行
```
/console clear-history
```

## Cisco IOS
修改当前会话的命令历史数量
```
Router# terminal history size <0-256>
```

禁用当前会话的命令历史记录和回放
```
Router# terminal no history
```

修改特定 Terminal line 的命令历史数量
```
Router(config)# line vty 0 4
Router(config-line)# history size <0-256>
Router(config-line)# end
```

禁用特定 Terminal line 的命令历史记录和回放
```
Router(config)# line vty 0 4
Router(config-line)# no history
Router(config-line)# end
```