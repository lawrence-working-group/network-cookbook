+++
title = "JunOS 开票"
date = "2020-05-31"
tags = ["JunOS"]
+++

JunOS 开票流程如下

进入系统以后，使用

```shell
# 获取 "RSI" 文件
request support information | save /var/tmp/rsi.log
# 打包日志
file archive compress source /var/log/* destination /var/tmp/rsi-log.tgz
```

吧 `rsi.log` 和 `rsi-log.tgz` 提交 JNCIE 或 代理商
