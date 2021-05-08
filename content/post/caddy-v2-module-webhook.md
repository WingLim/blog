---
title: "Caddy V2 Webhook 模块"
date: 2021-05-08T13:07:28+08:00
slug: caddy-v2-module-webhook
tags: 
- Caddy
- Module
- Webhook
categories:
---



## 介绍

项目地址：[caddy-webhook](https://github.com/WingLim/caddy-webhook)

`caddy-webhook` 的流程分三步：

1. 初始化时拉取仓库到指定的本地路径，如果本地路径上已经存在仓库，则用 `fetch` 更新并切换到设定的分支。
2. 在仓库内执行用户设定的命令。
3. 在用户设定的网址路径上监听 webhook 请求，收到正确的请求后更新仓库并重复第2步。

## 使用方法

现在支持的 webhook 类型：

- github
- gitlanb
- gitee
- bitbucket
- gogs

### Caddyfile 格式

需要注意的是，webhook 这个 handler 需要放在 `route`，因为为了正确响应 webhook request，caddy-webhook 不会执行下一个 handler。

```
webhook [<repo> <path>] {
    repo       <text>
    path       <text>
    branch     <text>
    depth      <int>
    type       <text>
    secret     <text>
    command    <text>...
    key	       <text>
    username   <text>
    password   <text>
    token      <text>
    submodule
}
```

- repo - git 仓库的远程地址，支持 http、https、ssh
- path - git 仓库的本地路径
- branch - 分支名
- depth - 拉取时的深度。默认值为 `0`
- type - webhook 的类型。默认值为 `github`
- secret - 用户在 GitHub 等网站上设置的请求的密码，用于校验请求
- submodule - 是否加载子模块
- command - 要执行的命令
- key - 使用 ssh 方法获取 git 仓库时需要的 key 文件路径
- username - http 认证的用户名
- password - http 认证的密码
- token - GitHub 授权 Token



## 使用样例

`Caddyfile`

```
example.com

root www
file_server

route /webhook {
    webhook {
        repo https://github.com/WingLim/winglim.github.io.git
        path blog
        branch hugo
        command hugo --destination ../www
        submodule   
    }
}
```

