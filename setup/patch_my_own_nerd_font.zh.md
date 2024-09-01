---
title: "自行制作 nerd font"
date: 2023-03-18T15:06:08+08:00
lastmod: 2023-03-18T15:06:08+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["tips", "geek"]
description: "" #描述
weight: # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
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
**Nerd Fonts** 是一个使用大量字体图标来解决程序员在开发过程中缺少合适字体的问题的项目。它可以从流行的字体图标库中将大量外部字体引入待开发的项目中。

**Nerd Fonts**官方提供的`Fura Mono`字体将`r`修改成了`Fira Mono`的变种形式，个人不太喜欢，于是考虑自行打包。

同时，由于版权原因，未提供`monaco`字体的**Nerd Fonts**，因此也需要自行打包。

## patch `Fira Mono`
参照[Option 9: Patch Your Own Font](https://github.com/ryanoasis/nerd-fonts),下载[font patcher](https://github.com/ryanoasis/nerd-fonts/releases/latest/download/FontPatcher.zip)并解压，保证待打包字体位于解压之后的文件夹，然后执行
```sh
./font-patcher FiraMono-Regular.ttf -s -c --also-windows -ext otf
```

## patch `Monaco`
与`Fira Mono`类似，从[此处](https://github.com/Karmenzind/monaco-nerd-fonts)下载`Monaco`字体，然后执行：
```sh
./font-patcher Monaco.ttf -s -c --also-windows -ext otf
```