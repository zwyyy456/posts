---
title: "Nas 预组"
date: 2023-12-04T23:22:04+08:00
lastmod: 2023-12-04T23:22:04+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["nas", trick"]
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
## 功能简介

应该可以说是一台 AIO，利用 PVE 虚拟化多个系统，实现不同功能：

- 基于 Debian 的开发机，ssh 连接上去进行远程开发、编译；
- Nas，预计采用 TrueNAS SCALE；
- 其他服务，都通过 Docker 运行，考虑采用 FlatCar 这种容器化系统，也可能直接跑在 Debian 上。
    - qBittorrent;
    - Immich;
    - Memos;
    - Restic;
    - Plex Media Server
    - CloudDrive2
    - IYUUPlus
    - Alist
    - Rclone

## 硬件选择

### 联想 RD450x 

CPU 拟采用 E5 2680v4。

优点：硬件成本低，机箱 + 主板 + CPU 仅需 ￥600，且有极强的扩展性；

缺点：功耗高，不带硬盘的待机功耗约为 80W，满载功耗约为 200W，无 GPU 硬件转码功能；

### 梵隆机箱

主板可以选择 MATX 主板，CPU 考虑使用 12400？

优点：功耗相比服务器更低，12400 待机功耗约 30W，待机功耗大概能低 50W？考虑到上海电费，一年能省下 50 * 24 /1000 * 365 * 0.75 = 300 元人民币左右，可以使用 GPU 硬件进行转码；

缺点：扩展性不如服务器，且硬件成本高，大概能高出 3 年的电费；

## 结论

暂定，等毕业论文搞定之后考虑去 V2EX 和 CHH 上发帖咨询之后再决定。
2055 427 1038

