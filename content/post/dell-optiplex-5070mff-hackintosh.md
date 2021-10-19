---
title: "戴尔 5070MFF 黑苹果体验"
author: WingLim
description: 一台充满乐趣的小主机
date: 2021-03-10T19:09:40+08:00
slug: dell-optiplex-5070mff-hackintosh
tags:
- Hackintosh
- Dell
categories:
- Tech
---

EFI开源托管在GitHub：[Dell-Optiplex-5070mff-Hackintosh](https://github.com/WingLim/Dell-Optiplex-5070mff-Hackintosh)

![Tested in Big Sur 11.2.3](https://cdn.jsdelivr.net/gh/WingLim/assets@master/images/20210315163415.png)

## 配置介绍

硬件配置：

- 准系统: [Dell OptiPlex 5070 Micro Form Factor](https://www.dell.com/en-us/work/shop/desktops-all-in-one-pcs/optiplex-5070-micro/spd/optiplex-5070-micro)
- CPU: [Intel® Core™ i5-9500T Processor](https://ark.intel.com/content/www/us/en/ark/products/191052/intel-core-i5-9500t-processor-9m-cache-up-to-3-70-ghz.html)
- 核显: Intel® UHD Graphics 630
- 内存: 8GB DDR4 2666 * 2 双通道
- 硬盘: KIOXIA RC10 NVME SSD 500G
- Wi-Fi & Bluetooth: DW1820A
- 声卡: Realtek ALC255(3234)
- 板载网卡: Intel I219-LM7

接口配置：

前面板：

- 通用音频接口
- 有线音频输出
- Type C(USB3.1 Gen2 PowerShare) 注：不是雷电接口
- Type-A USB接口(USB 3.1 Gen1 PowerShare)

后面板：

- RJ-45网线接口
- Type-A USB接口(USB 3.1 Gen1) * 4
- DP接口 * 2

## 正常功能

- CPU睿频
- 核显加速
- Airdrop & Airplay & Handoff
- 所有USB接口
- 有线及无线网
- 扬声器 & 通用音频接口 & 有线音频输出
- 睡眠

## 安装前准备

安装黑苹果前需要使用GRUB，将关闭CFG锁和设置预分配的DVMT内存到64M。

将GRUB的`EFI`文件夹放入U盘根目录，通过U盘启动，输入如下两行命令：

```shell
// Disable CFG lock
setup_var 0x5BE 0x00

// Set Pre-Allocated DVMT to 64M
setup_var 0x8DC 0x02
```

系统安装镜像请到[黑果小兵的部落阁](https://blog.daliansky.net/)下载

## 遇到的问题

1. 核显驱动

   戴尔主板的显示总线在`1号`和`2号`，注入核显型号后还需要注入总线ID，因为主板上只有两个DP接口，所以把`con2`屏蔽掉了。但5070MFF主板上有接口可以扩展出新的视频输出，如果有需要需要自己测试。

   ![核显驱动设置](https://cdn.jsdelivr.net/gh/WingLim/assets@master/images/20210314204729.png)

2. DW1820A的Wi-Fi驱动

   使用该网卡时需要屏蔽4个pin引脚，屏蔽后插入即可直接驱动Wi-Fi

   具体需要屏蔽的pin引脚请看，使用胶带进行屏蔽即可：[DW1820A/BCM94350ZAE/BCM94356ZEPA50DX插入的正确姿势](https://blog.daliansky.net/DW1820A_BCM94350ZAE-driver-inserts-the-correct-posture.html)
