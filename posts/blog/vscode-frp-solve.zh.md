---
title: "解决 vscode remote 无法通过 frp 连接到内网机器的问题"
date: 2023-09-11T14:58:42+08:00
lastmod: 2023-09-11T14:58:42+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: [""]
tags: [""]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---
## 问题描述
本人有一台无公网 ip 的 Debian 主机，为了能够远程连接上去写代码，因此配置好了 frp，frp 服务商是公益性质的 mossfrp，之前都正常运行，vscode remote 可以正常连接上去使用，体验与本地开发基本一致。然而昨天，突然就莫名其妙地出现了问题，当我 vscode 连接上去之后，会很快断开连接，然后显示重连失败，如下图所示：

![132slqjDmOXzKCF](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065729.png)

在 vscode 也看不到相关的日志输出。试过重启大法，依旧无效。

## 问题解决

本着排除法的原则，我借同学的 windows 电脑上的测试了一下，在它的电脑上，vscode remote 是能正常通过 frp 连接到我的内网机器的，并且不会出现自动断开的情况。此时，我推测可能是 vscode 或者操作系统的问题（这个概率不大）。

后来在 vscode 的 github 的 issue 中一搜，已经有相关的 [issue](https://github.com/microsoft/vscode-remote-release/issues/8926) 了，是 vscode remote 这个扩展在 0.106.1 版本的问题，可以通过降级插件或者切换到最新的预览版解决，又或者是将 `remote.SSH.useExecServer` 设置为 false。

