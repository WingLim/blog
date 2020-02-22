---
title: Hexo 主题 Typing
urlname: hexo-theme-typing
date: '2019-11-15 21:51:31'
updated: '2019-11-16 20:44:38'
categories:
  - 网站
tags:
  - Hexo
  - Theme
typora-root-url: ../
---

这两天将博客迁移到 Hexo ，部署在阿里云上，备份放在 GitHub 上，然后自己动手给 Hexo 写了一个主题，名为 Typing，希望更加专注在写文章上。

<!--more-->

功能介绍：

1. 自定义 favicon

2. 自定义 keywords，优化SEO

3. 自定义 footer 信息

4. 使用 [DisqusJS](<https://github.com/SukkaW/DisqusJS>) 实现评论
5. 自定义 HTML 的 lang，不用 Hexo 的配置文件来定义是因为换成 zh-cn 后，Moment.js 生成的时间格式是 Chinese Style，比如 “十一月” 这种，然后我自己更喜欢英文的月份，所以在主题配置文件中加了这个选项

使用方式：

```shell
git clone https://github.com/WingLim/Hexo-Theme-Typing.git themes/Typing
```

修改网站根目录下的`_config.yml`

```yaml
theme: Typing
```



GitHub Repo：[Hexo-Theme-Typing]( https://github.com/WingLim/Hexo-Theme-Typing )

TODO：

- [x] Optimizate SEO

- [x] Comment

