---
title: frp 一键安装脚本
date: '2018-12-26 19:41:22'
slug: frp-install-script
tags:
  - frp
  - script
categories:
  - 折腾
---

 在社团的活动室发现了一台闲置的服务器，服务器用的是学校的网络，没有公网IP，只能局域网访问，所以用 frp 搭了个内网穿透，然后为了方便部署服务端，就写了个一键安装脚本。

<!--more-->



## 介绍

脚本会自动从 [frp](https://github.com/fatedier/frp) 中获取最新的 release 

注意：目前只有安装服务端的功能，并且只保留 frps 的二进制文件

脚本创建一个 frps.service 来控制 frps 的开关，默认开启开机自启

frps 安装在 ` /usr/local/frp `

配置文件在 ` /usr/local/frp/frps.ini `



## 使用方法

### 安装

```shell
wget https://raw.githubusercontent.com/WingLim/frp_install_script/master/frps.sh
sudo chmod +x frps.sh
sudo ./frps.sh 2>&1 | tee frps.log
```



### 使用命令

```shell
systemctl start frps # 启动 frps
systemctl stop frps # 停止 frps
systemctl status frps # 关闭 frps
```



TODO：

- [ ] frp 一键安装客户端脚本