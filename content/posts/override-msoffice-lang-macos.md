+++
title = "强制 macOS 上的 Microsoft Office 套件使用指定语言（与系统语言不同）"
date = "2020-01-17"
tags = ["macOS", "Microsoft Office", "i18n"]
+++

命令行/终端里输入这样几条即可

```console
defaults write com.microsoft.Excel AppleLanguages '("zh-CN")'
defaults write com.microsoft.Word AppleLanguages '("zh-CN")'
defaults write com.microsoft.Powerpoint AppleLanguages '("zh-CN")'
```