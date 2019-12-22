# 本地用户的配置

## 适用于 Cisco IOS/IOS XE

```
username <用户名> privilege 15 algorithm-type scrypt secret <用户密码>
```

其中 privilege 后面的数字 15 代表权限级别，15 对应 Cisco IOS 中的最高管理权限。
scrypt 是加密方式，是 Cisco IOS 中目前最安全的密码散列算法。