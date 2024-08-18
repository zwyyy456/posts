---
title: "solve the problem of downloading assets from github"
date: 2022-10-13T21:38:19+08:00
lastmod: 2022-10-13T21:38:19+08:00 #更新时间
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
## Description
After version 2021a, in order to reduce the file size, *Webots* set resource files such as textures and sounds up for network download by [github](https://raw.githubusercontent.com/cyberbotics/webots/R2022b/). However, for well-known reasons, github is very inaccessible in China, so there will be errors like:
```
ERROR: Error downloading EXTERNPROTO 'StraightStairs': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/objects/stairs/protos/StraightStairs.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'Spot': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/robots/boston_dynamics/spot/protos/Spot.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'Roughcast': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/appearances/protos/Roughcast.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'ThreadMetalPlate': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/appearances/protos/ThreadMetalPlate.proto', error code: 1: Connection refused
```
which means you can't download resource files from github.

## Solution
We can change the `network` preferences in Webots.

Click `preferences->network`, in the `proxy` column, check `SOCKS v5`, fill in `127.0.0.1` for `Hostname` and `7890` for `Port`.

Attention:First of all your own computer should have a proxy, and the port is related to your own settings.