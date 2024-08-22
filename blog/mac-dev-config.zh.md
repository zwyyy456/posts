---
title: "Mac 开发环境配置"
date: 2023-11-17T17:00:43+08:00
lastmod: 2023-11-17T17:00:43+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["mac", "dev"]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---
## 前言

本文内容主要是我对 Mac 的所做的配置修改的一些记录。后续会随着时间更新，免得忘记自己对 Mac 做过什么修改了。

## 安装 Command Line Tools for Xcode

安装包下载 [链接](https://developer.apple.com/download/all/)

## 安装 Homebrew

执行 `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"` 即可。网络环境已自带科学。

大名鼎鼎的 Homebrew 无需多说，通过 Homebew 我安装了以下软件：

![4XJ4iQ](https://pic-upyun.zwyyy456.tech/uPic/4XJ4iQ.png)

总的原则大概就是能用 homebrew 安装的都是使用 homebrew 安装，例如 Edge、VScode 等。

要注意的是，Homebrew 安装软件时，将软件分成了 Formulae 和 Casks 两大类，简单来说就是，有 gui 的就是 Casks，没有的就是 Formulae，安装带 GUI 的软件，例如 Neovide 时，安装指令为 `brew install neovide --cask`。

Homebrew 还可以用于安装字体，例如执行 `brew install font-fira-mono-nerd-font --cask` 即可安装 FiraMono 字体。


## 安装 Neovim

执行 `brew install neovim` 即可，然后 `cd ~/.config/` 执行 `git clone git@github.com:zwyyy456/nvim_config.git && mv nvim_config nvim`，之后终端中运行 `nvim`，就会自动配置 LazyVim。

详见 [Neovim 的配置与使用](https://blog.zwyyy456.tech/zh/posts/blog/neovim_tutorial/)。

## 安装 miniconda

直接执行 `brew install miniconda` 即可。

`miniconda` 应该会被安装到 `/opt/homebrew/Caskroom/miniconda` 目录下。

执行 `nvim ~/.zshrc`，添加如下内容：

```sh
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/homebrew/Caskroom/miniconda/base/bin/conda' 'shell.zsh' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/homebrew/Caskroom/miniconda/base/etc/profile.d/conda.sh" ]; then
        . "/opt/homebrew/Caskroom/miniconda/base/etc/profile.d/conda.sh"
    else
        export PATH="/opt/homebrew/Caskroom/miniconda/base/bin:$PATH"
    fi
fi
unset __conda_setup
```


