---
title: "解决webots无法从github下载外部资源的问题"
date: 2022-10-13T10:25:10+08:00
lastmod: 2022-10-13T10:25:10+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: [""]
tags: [""]
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
## 问题描述
*Webots*在2021a版本后，为了缩小文件大小，将纹理、声音等资源文件设置成网络下载，即需要通过[github](https://raw.githubusercontent.com/cyberbotics/webots/R2022b/)进行下载，然而由于众所周知的原因，github在国内的可访问性非常差，因此会出现
```
ERROR: Error downloading EXTERNPROTO 'StraightStairs': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/objects/stairs/protos/StraightStairs.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'Spot': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/robots/boston_dynamics/spot/protos/Spot.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'Roughcast': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/appearances/protos/Roughcast.proto', error code: 1: Connection refused
ERROR: Error downloading EXTERNPROTO 'ThreadMetalPlate': Cannot download 'https://raw.githubusercontent.com/cyberbotics/webots/R2022b/projects/appearances/protos/ThreadMetalPlate.proto', error code: 1: Connection refused
```
即无法从github上把资源文件下载下来。

## 解决方案
webots的`preferences`中有关于代理的设置(这也就是为什么系统代理没有生效)，点击`preferences->network`，在`proxy`那一栏，勾选`SOCKS v5`, `Hostname`填入`127.0.0.1`，`Port`填入`7890`。
注:首先你自己的电脑要有代理，端口(Port)与你自己的设置有关系。
