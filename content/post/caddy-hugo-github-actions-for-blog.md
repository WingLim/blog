---
title: "Caddy2 + Hugo + Github Actions 的自动化部署博客方案"
date: 2021-05-08T18:35:53+08:00
slug: caddy-hugo-github-actions-for-blog
tags:
- Caddy
- Hugo
- GitHub Actions
categories:
- Tech
- DevOps
---



我的博客部署流程如下：

1. 写文章并推送到 `username.github.io` 仓库的 `hugo` 分支。
2. GitHub Actions 自动构建并推送到 `main` 分支。
3. GitHub 发送 webhook 请求到自有服务器，服务器拉取更新。



不使用 GitHub Page 的原因主要是在国内访问太慢，而且有服务器闲置，正好用来部署博客。

而使用 GitHub Actions 先构建推送到 `main`，然后再在服务器上拉取的原因是可以在 GitHub Page 上有一个备份，服务器出现故障时可以先 302 重定向到 GitHub Page，解决故障后切换回来。



为了实现这个流程，在服务器上需要用到一个服务：Caddy

Caddy 是基于 go 编写的 web 服务器，相比于 nginx 和 apache 的优点就是能自动申请 SSL 证书，自动更新证书。

当然，有人会说这类文章网上已经有很多了，为什么还要重复再写一篇。一个重要的原因是网上的的文章都是基于 Caddy V1 版本，而在 Caddy 更新到 V2 版本后，之前的插件都已经失效了。

本着用新不用旧的原则，我也将 Caddy 更新到 V2，但也因为这样需要重新配置第 3 步的部署流程。

为了实现第 3 步，我给 Caddy 写了一个模块：[caddy-webhook](https://github.com/WingLim/caddy-webhook)，下面通过具体的步骤来展示如何使用这个模块。



### 建立仓库

建立一个 `username.github.io` 的仓库会自动配置 GitHub Page，并且可以通过直接访问 `username.github.io` 来访问到 `main` 分支中的静态页面。

克隆仓库并进入

```bash
git clone username.github.io
cd username.github.io
```

新建并切换到 `hugo` 分支

```bash
git checkout -b hugo
```

在当前目录创建一个 hugo 站点

```bash
hugo new site .
```

然后就可以在 `hugo` 分支中撰写文章了



### 使用 GitHub Actions

创建 github workflows 文件夹

```bash
mkdir -p .github/workflows
```

进入 workflows 文件夹并新建 `hugo` 工作流

```bash
cd .github/workflows
touch hugo.yml
```

`hugo.yml` 内容如下：

因为在 Hugo 站点中，大多数人都是使用 `submodule` 来配置主题，所以我们需要使用 `checkout@v2` 的 `submodules: 'recursive'` 来获取主题，否则在构建博客时会无法找到主题模版而无法生成静态页面。

同时推荐使用 `extended` 版本的 Hugo，附带了 scss 的功能，以免使用的主题没有提供编译后的 css 文件。

`secrets.GITHUB_TOKEN` 是 GitHub Actions 中自带的，无需再手动配置。

```yaml
name: Deploy Hugo to Github Pages

on:
  push:
    branches:
      - hugo

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
           submodules: 'recursive'

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.83.1'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: main
```



### 部署 Caddy

创建文件夹用于保存 Caddy 的文件数据

```bash
mkdir -p caddy/data
```

初始化日志及配置文件

```bash
cd caddy
touch access.log
touch Caddyfile
```

使用 docker 进行部署，`winglim/caddy` 是我构建的包含了 `caddy-webhook` 模块的镜像。

```bash
docker run -itd \
    -p 80:80 -p 443:443
    -v $PWD/Caddyfile:/etc/caddy/Caddyfile \
    -v $PWD/access.log:/var/log/access.log \
    -v $PWD/data:/data\
    winglim/caddy
```

如果不想使用 docker 进行部署的话，可以自己手动编译一个带 `caddy-webhook` 模块的 caddy

```bash
# 注意 go install package@tag 只在 go 1.16 及以上版本才支持
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
# 低于 go 1.16 版本请使用 go get
go get -v github.com/caddyserver/xcaddy/cmd/xcaddy@latest

xcaddy build \
	--with github.com/WingLim/caddy-webhook
```



`Caddyfile` 内容如下：

```Caddyfile
example.com {
  tls yourmail@example.com
  encode zstd gzip

  log {
    output file /var/log/access.log
  }

  root blog
  file_server
  
  handle_errors {
  	@404 {
  		expression {http.error.status_code} == 404
  	}
  	handle @404 {
  		rewrite * /404.html
  		file_server
  	}
  }
  
  route /webhook {
    webhook {
      repo https://github.com/username/username.github.io.git
      branch main
      path blog
      secret yoursecret
    }
  }
}
```



最后，我们就实现了文章开头所说的工作流程，剩下的就是写一些有价值的文章了。