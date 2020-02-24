---
title: 小米笔记本PRO折腾黑苹果记
date: '2019-11-27 23:13:45'
slug: mi-book-pro-with-hackintosh
tags:
  - hackintosh
  - xiaomi
categories:
  - 折腾
typora-root-url: ../
---

在 Windows 下写代码，配置环境总有点蛋疼，而 Linux 的桌面环境又经常会有些bug，于是就给自己笔记本装了个 Hackintosh，不过在笔记本上并不是很完美，记录一下一些解决办法。

<!--more-->

## WIFI及蓝牙

#### 使用的硬件

1. COMFAST 811AC
2. 绿联蓝牙接收器（CSR8510）

因为 macOS 没有 Intel 的 WIFI 驱动，所以小米笔记本板载 WIFI 没办法使用，然后我两个 m.2 口都装了硬盘，没办法用拆机网卡，所以只能使用 USB 网卡。

而且蓝牙驱动也有问题，不能热加载，就是说想连蓝牙得先在 Windows 上连接，然后重启进入 Hackintosh 才能连接。

### 该软件包与此版本的 macOS 不兼容

在安装 WIFI 驱动的时候会出现这个问题：![WiFi Driver Problem](/images/wifi-drive.png)

这是因为在 macOS Catalina 中，对系统读写权限加了限制，需要允许安装任意来源的 App，并挂载根目录，才能安装。

```shell
sudo spctl --master-disable # 允许任意来源
sudo mount -uw / # 挂载根目录
killall Finder # 杀掉Finder
```

### 禁用自带蓝牙

想使用 USB 蓝牙接收器，需要先把笔记本自带的禁用。

我使用的是 [XiaoMi-Pro-Hackintosh](https://github.com/daliansky/XiaoMi-Pro-Hackintosh) 这个 EFI，只需要将 `SSDT-USBBT.aml` 替换掉`/EFI/CLOVER/ACPI/patched/SSDT-USB.aml` 即可禁用。

需要安装驱动的 USB 蓝牙接收器参考这里：[蓝牙解决方案]([https://github.com/daliansky/XiaoMi-Pro-Hackintosh/wiki/%E8%93%9D%E7%89%99%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/wiki/蓝牙解决方案))