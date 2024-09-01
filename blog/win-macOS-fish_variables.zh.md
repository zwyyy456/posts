---
title: "解决 macOS 上的 fish 出现的 'et_color Unkown color ' 问题"
date: 2023-12-18T11:14:37+08:00
lastmod: 2023-12-18T11:14:37+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["trick", "fish"]
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
## 问题简介

我主要使用的 shell 是开箱即用的 fish，fish 的配置文件位于 `~/.config/fish` 中，整个 `.config` 目录使用 Filen 实现在 macOS 和 Windows 之间的同步，并且使用 Git 进行版本管理。

当我昨天在 Windows 下修改了 `.config/nvim` 目录中的文件，将修改 push 到 GitHub 之后，今天在 Mac 上将修改 pull 到本地之后，fish 就出现了 `'et_color: Unkown color '` 问题，如下图所示：

![aveB6FUwRyYM7Lx](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065726.png)

## 解决方案

检查 `~/.config/fish/fish_variables` 文件，与 Filen 中记录的历史版本进行对比，发现每一行结尾都多了 `\x0d`，经查阅资料， `\0d` 表示回车，即 `\r`，而 mac 默认的换行标识为 `\n`。

删掉所有的 `\x0d` 之后就正常了，而问题出现的原因也很清楚了，在编辑文件（也可能是 Git 同步）的时候，`fish_variables` 文件的换行被从 `\n` 替换成了 `\r\n`，于是，就出现了上述的问题。

> 在我记忆中，我在 Windows 下应该是没有修改过 `fish_variables`文件的。

> 后续需要配置一下 Windows 下 Git 的换行符设置问题。

