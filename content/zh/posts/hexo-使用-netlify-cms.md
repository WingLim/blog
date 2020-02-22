---
title: Hexo 使用 Netlify CMS
date: '2019-12-16 05:17:32'
updated: '2019-12-16 05:17:32'
tags:
  - Hexo
  - Netlify
categories:
  - 网站
urlname: hexo-netlify-cms
typora-root-url: ../
---
想给 Hexo 添加一个后台管理页面，可以在浏览器上开箱即用。然后发现了JAMstack（JavaScript+APIs+Markup），通过 JavaScript 和 API 在前端直接增删改查，再触发 CI 来构建、部署。[Netlify CMS](https://www.netlifycms.org/) 就是这样一个 Serverless 的 CMS。

<!--more-->

## 配置插件

首先安装 [hexo-netlify-cms](https://github.com/JiangTJ/hexo-netlify-cms)

```shell
npm i hexo-netlify-cms --save
```

启用插件并设置自定义配置文件：

```yaml
netlify_cms:
  config_file: netlify-cms.yaml
```

在根目录下新建 `netlify-cms.yaml` 

```yaml
backend:
  name: github # 认证方式
  repo: owner/repo
  branch: master

media_folder: source/images # 媒体源文件夹
public_folder: /images # 生成后的媒体文件夹
publish_mode: editorial_workflow # 发布模式

# 配置自动生成 collection
auto_generator:
  post: 
    all_posts:
      #enabled: true
      label: "Post"
      preview_path: "/posts/{{fields.urlname}}" # 预览地址
      folder: "source/_posts" # 文章源文件夹
      create: true # 允许用户新建文章
      editor:
        preview: true
  page: 
    enabled: true
    config:
      label: "Page"
      delete: false
      editor:
        preview: true

# 设置全局表单
global_fields:
  over_format: true
  default:
    - {label: "Title", name: "title", widget: "string"}
    - {label: "Publish Date", name: "date", widget: "datetime", dateFormat: "YYYY-MM-DD", timeFormat: "HH:mm:ss", format: "YYYY-MM-DD HH:mm:ss", required: false}
    - {label: "Update Date", name: "updated", widget: "datetime", dateFormat: "YYYY-MM-DD", timeFormat: "HH:mm:ss", format: "YYYY-MM-DD HH:mm:ss", required: false}
    - {label: "Tags", name: "tags", widget: "list", required: false}
    - {label: "Categories", name: "categories", widget: "list", required: false}
    - {label: "Permalink", name: "urlname", widget: "string"}
    - {label: "Body", name: "body", widget: "markdown"}
  #post:
  #page:
```

## 自建 GitHub Oauth 服务

因为我的博客是用 Travis-CI 自动部署到阿里云上，所以没办法用 Git-Gateway 来身份认证。

而 GitHub Oauth 认证需要服务器验证，解决方法有两个：

1. Netlify 提供
2. 自己搭建

在使用 Netlify 的接口时出现无法认证的问题（应该是要在 Netlify 里面绑定域名），耦合度太高了不是很喜欢，而且速度有时候很慢，所以自己搭了一个。

项目地址：[netlify-cms-oauth-provider-python](https://github.com/WingLim/netlify-cms-oauth-provider-python)

首先要修改配置文件：

```yaml
backend:
  name: github
  repo: owner/repo
  branch: master
  base_url: https://app.limxw.com # GitHub Oauth 服务地址
```

使用 Docker 部署：

1. 使用配置文件：.env

```yaml
OAUTH_CLIENT_ID=12345
OAUTH_CLIENT_SECRET=balabala
REDIRECT_URL=https://your.server.com/callback
```

REDIRECT_URL：如果需要回调地址与 Oauth 应用程序配置中提供的不同，设置此项。

```shell
docker run -itd \
    -v /your/path/.env:/usr/src/app/.env \
    -p 7676:80 \
    winglim/netlifycms-oauth
```

2. 使用环境变量

```shell
docker run -itd \
    -e OAUTH_CLIENT_ID=12345 OAUTH_CLIENT_SECRET=balabala \
    -p 7676:80 \
    winglim/netlifycms-oauth
```

之后只需要用 nginx 或者 caddy 等软件将域名反向代理到 ip:7676 即可（也可以修改成其他端口
