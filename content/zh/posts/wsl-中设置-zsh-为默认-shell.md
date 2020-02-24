---
title: WSL 中设置 zsh 为默认 SHELL
date: '2018-12-29 02:15:00'
updated: '2019-12-19 08:20:47'
tags:
  - WSL
  - Linux
categories:
  - 折腾
slug: wsl-use-zsh
typora-root-url: ../
---
最近在用 Oh My ZSH! ，然后在 WSL(Windows Subsystem for Linux)上也安装了 zsh 和 Oh My ZSH! 但是在设置默认 SHELL 时出现了问题。

<!--more-->

Linux 下设置默认 SHELL 方法如下：
```shell
chsh -s /bin/zsh
```

但重新在 CMD/POWERSHELL 上进入 WSL ，默认的 SHELL 还是 bash ，需要手动执行 ``$ zsh``才能进入。

于是在 Google 上查了一下，发现在 Microsoft 的 Github 上面有一个提交 Bug 的 Repository：[Microsoft/WSL](https://github.com/Microsoft/WSL) 上有一个 issue：

[can't change default shell #477](https://github.com/Microsoft/WSL/issues/477)

出现这个问题的原因是在启动 WSL 时没有执行 login 相关的组件，而这些组件和设置默认 SHELL 有关。
> We don't run login which is the component that normally sets those things up.

## 解决方法
打开 ``~./bashrc`` 添加下列代码进去并保存即可。

```shell
[[ $- == *i* ]] && $(command -v zsh) || echo "ZSH is not installed"
```

命令的具体解释在这里：[https://github.com/Microsoft/WSL/issues/477#issuecomment-441164103](https://github.com/Microsoft/WSL/issues/477#issuecomment-441164103)
