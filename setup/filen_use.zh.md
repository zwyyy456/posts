---
title: "使用 Filen 同步文件夹"
date: 2023-11-28T15:36:08+08:00
lastmod: 2023-11-28T15:36:08+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["trick"]
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
## Filen 介绍

这里直接引用官网的介绍："Zero knowledge end-to-end encrypted cloud storage, redefined"。

即这是一个端对端加密的同步盘，类似 Dropbox 和坚果云，不同于百度网盘、115、阿里云盘这样的资源盘，支持文件历史版本。

通过他人的邀请链接注册可以获得 20G 的初始容量，之后邀请 3 个人注册可以再获得 30G 的容量，就同步来说，50G 其实也够了，我趁着黑五促销，又买了 100G 的永久容量。

目前看来，它的同步功能做得不如坚果云，不支持 webdav，也没有云桥功能（虽然这个功能我用不到）但是坚果云不付费的限制比较多，而 Filen 付费与否只与容量相关。

此外，Filen 对我而言有一个杀手级别的功能，那就是支持设置 `.filenignore`，即采用类似 `gitignore` 的语法，不同步某些特定文件，例如 build 的产物等。

## 文件夹同步设置

前文提到，Filen 是一个同步盘，我的用途也是实现 mac 和 windows 笔记本之间的同步，主要包括配置文件、代码文件以及博客文本文件等。

本来 onedrive 的同步功能也还可以，但是不支持“指定特定文件夹不同步”让他在同步带 `.git` 的代码文件夹时，非常容易出问题。

同步文件夹的设置如下图：

![pIRWJzrsCok7UwQ](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065718.png)

点击 `create one` 就会让你选择一个本地的文件夹，选定好之后，`settings->syncs` 会多出一栏，点击右边的设置图标，即可配置 `.filenignore`，语法同 `.gitignore`，`sync mode` 一般选择 `Two Way`。然后点击 `select remote location`，可以选择要同步到云盘中的哪个文件夹，选择好之后就可以开始同步了。

## 选择同步的文件夹

目前，mac 上我同步了这些文件夹：

![N4JStkHjhQiP1Il](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065721.png)

尽管除了 `.ssh` 和 `Pictures` 文件夹之外，其他文件夹都有通过 git 和进行 GitHub 进行版本管理与同步，但是难免有忘记 `commit` 和 `push` 的情况。例如，假设我在 mac 上对代码进行了修改，但是忘记 push 到 GitHub 了，然后回到了宿舍，此时宿舍的 Windows 笔记本无法获取到 mac 上对代码的修改，但有了 Filen 同步就不一样了，尽管 git 的状态与 mac 不一致，但源文件的修改是一致的，Windows 修改了之后，再 push 到 GitHub 上即可（实际上不 push 也没啥问题）。

## 文件组织

### mac 

- 博客都存放在 `~/Documents/blog/` 目录下，会被自动上传到 iCloud 来进行备份；
- 代码文件分为两类：
    - 可以直接在 mac 上运行的代码，例如 python，leetcode 直接存放在 `~/Documents/code` 目录下，会被自动上传到 iCloud 进行备份；
    - 需要 Linux 环境才能运行的代码，例如 webserver，存放在 OrbStack 的虚拟机的 `~/code` 目录下；

### Windows

- 博客都存放在 `~/OneDrive/Documents/blog/` 目录下，会被自动上传到 OneDrive 来进行备份；
- 代码文件分为两类：
    - 可以直接在 Windows 上运行的代码，例如 python，leetcode 直接存放在 `~/OneDrive/Documents/code` 目录下，会被自动上传到 OneDrive 进行备份；
    - 需要 Linux 环境才能运行的代码，例如 webserver，存放在 WSL 的虚拟机的 `~/code` 目录下；

> Filen 无法同步 WSL 中的文件夹，因此代码文件需要从 GitHub pull，可以先在 `~/OneDrive/Documents/code/xxx` 中 push 到 GitHub，再从 WSL 中 pull。

## Filen 的缺点

当然，Filen 也不是十全十美的，依旧存在一些问题：

1. 客户端基于 Electron 开发，内存占用高，即使没有任何活动，内存占用也可能接近 600M；
2. 不支持 webdav；
3. 客户端无法直接管理云盘的文件夹，必须在网页端或者手机端管理；
4. 无法同步 WSL 中的文件夹，但是 orbstack 创建的虚拟机中的文件夹似乎可以；



