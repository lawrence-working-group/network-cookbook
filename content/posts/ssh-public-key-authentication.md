+++
title = "网络设备 SSH 使用公钥进行用户认证的配置"
date = "2020-05-23"
tags = ["SSH"]
+++

## Cisco IOS

1. 使用以下命令之一，列出 SSH 公钥对的哈希
    ```
    ssh-add -l -E md5
    ssh-keygen -E md5 -lf ~/.ssh/id_rsa.pub
    ```

2. 再将上述命令的结果使用管道输出到下列命令
    ```
    sed -e 's/.*MD5:\([0-9a-f:]*\) \(.*\)(.*)/key-hash ssh-rsa \1 \2/' -e 's/://g
    ```
    上述命令提取出每一条记录的 MD5 哈希后，生成可以复制的思科 IOS 命令。形如：`key-hash ssh-rsa <yourkeyhash> YOUR-COMMENT`
    
    空格后的部分是可以自定义的关于此公钥对哈希的描述。

3. 为单个用户配置对应的 SSH 公钥对哈希
    ```
    Router(config)#ip ssh pubkey-chain
    Router(conf-ssh-pubkey)#username <username>
    Router(conf-ssh-pubkey-user)#key-hash ssh-rsa <yourkeyhash> YOUR-COMMENT
    ```

4. 在不退出当前会话的情况下，使用另一个 ssh 会话尝试与远程设备进行用户认证。

## Cisco ASA
1. 以 RFC 4716 格式输出现有 SSH 公钥
    ```
    ssh-keygen -e -f ~/.ssh/id_rsa.pub
    ```

2. 在 ASA 配置界面输入以下命令，然后粘贴前一条输出的公钥，结束后换行输入 quit，完成公钥配置。
    ```
    ciscoasa(config)# username <username> attributes
    ciscoasa(config-username)# ssh authentication pkf
    ```

3. ASA 命令行提示：`INFO: Import of an SSH public key formatted file completed successfully.`

4. 在不退出当前会话的情况下，使用另一个 ssh 会话尝试与远程设备进行用户认证，并保存当前配置。
