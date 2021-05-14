+++
title = "Aruba Console 定义"
date = "2021-05-15"
tags = ["HPE", "Aruba"]
+++

# 4 PIN header

如果你拿到的型号是 1xx / 2xx / 3xx 等型号的 Aruba AP，那么的 Console 一组 UART PINOUT

按照 Console 文字方向的线序依次为

| #PIN | Name |
| ---: | ---- |
|    1 | GND  |
|    2 | TX   |
|    3 | RX   |
|    4 | 3.3V |

- <https://community.arubanetworks.com/community-home/digestviewer/viewthread?MID=17312#bm8b7a050f-bbbd-49c2-b31e-8d8949de581f>
- <https://support.arubanetworks.com/Documentation/tabid/77/DMXModule/512/Command/Core_Download/Default.aspx?EntryId=25047>

# microUSB

如果你拿到的型号是 5xx 等型号的 Aruba AP，那么的 Console 一个 microUSB 的 UART

![microUSB PINOUT](https://i.imgur.com/ol0sm2L.jpeg)

线序依次为

| #PIN | Name |
| ---: | ---- |
|    1 | 3.3V |
|    2 | TX   |
|    3 | RX   |
|    4 | GND  |
|    5 | GND  |

- <https://support.arubanetworks.com/Documentation/tabid/77/DMXModule/512/Command/Core_Download/Default.aspx?EntryId=25048>
