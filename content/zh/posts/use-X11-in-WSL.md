---
title: "在 WSL 中使用 X11 Forwarding"
date: 2020-02-25T00:35:53+08:00
slug: use-x11-in-wsl
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
tags:
- WSL
- Qemu
series:
-
categories:
- 折腾
image:
typora-root-url: ..\..\..\static\
---

最近在使用 WSL 作为开发环境，但有些图形化软件无法打开，例如 Qemu，通过 X11 Forwarding 来使得 Qemu 正常显示。

<!--more-->

## 什么是 X 协议

X 协议由 X server 和 X client 组成：

X server 管理主机上与显示相关的硬件设置（如显卡、硬盘、鼠标等），它负责屏幕画面的绘制与显示，以及将输入设置（如键盘、鼠标）的动作告知 X client。

X client (即 X 应用程序) 则主要负责事件的处理（即程序的逻辑）。

## 什么是 X11 Forwarding

许多时候 X server 和 X client 在同一台主机上，这看起来没什么。但是， **X server 和 X client 完全可以运行在不同的机器上，只要彼此通过 X 协议通信即可。**

于是，我们就可以做一些“神奇”的事情，在本地显示 (X server)运行在服务器上的 GUI 程序 (X client)。这样的操作可以通过 SSH X11 Forwarding 来实现。X11 中的 X 指的就是 X 协议，11 指的是采用 X 协议的第 11 个版本。

## 在 Windows 上安装 X server

1. 是选择使用 [MobaXterm](https://mobaxterm.mobatek.net/) ，这个软件集成了 X server

2. 安装 [VcXsrv Windows X Server](https://sourceforge.net/projects/vcxsrv/files/vcxsrv/)

安装完 VcXsrv 后，打开 XLaunch ：

![VcXsrc Setting](/images/VcXsrc-settings1.png)

选择 Multiple windows ,则只会传输软件窗口本身，而其他三个则会传输整个桌面环境。其余选项选择默认即可。

然后在 `.bashrc` 或 `.zshrc` 中添加显示器地址：

```bash
export DISPLAY=localhost:0
```

同时确保在 `/etc/ssh/sshd_config` 中有 `X11Forwarding yes` 且未被注释掉。



## 测试

```bash
# 安装 x11-apps
sudo apt install x11-apps
# 打开 xclock
xclock
```

若出现如下窗口则开启成功：

![xclock](/images/xclock.png)