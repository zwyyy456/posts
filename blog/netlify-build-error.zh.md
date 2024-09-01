---
title: "解决 Netlify 部署 Hugo 静态博客时的 `fatal: remote error: upload-pack: not our ref` 问题"
date: 2023-12-18T12:46:49+08:00
lastmod: 2023-12-18T12:46:49+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["hugo", "trick", "netlify"]
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
## 问题描述

我的其中一个 Hugo 博客采用了 PaperMod 主题，另一个是 hugo-coder 主题，都是通过 Netlify 部署，昨天晚上发现，Netlify 显示 `paper.191000.xyz` 对应的站点部署失败。

问题日志显示：

```txt
12:26:05 PM: Waiting for other deploys from your team to complete. Check the queue: https://app.netlify.com/teams/zwyyy456/builds
12:26:20 PM: build-image version: 3ffff9df3d5419545acc1b673a54de348174406d (focal)
12:26:20 PM: buildbot version: 4613af4169363e3b38cfadfa4665d34cd1d1427b
12:26:20 PM: Fetching cached dependencies
12:26:20 PM: Starting to download cache of 106.8MB
12:26:22 PM: Finished downloading cache in 1.935s
12:26:22 PM: Starting to extract cache
12:26:23 PM: Finished extracting cache in 720ms
12:26:23 PM: Finished fetching cache in 2.7s
12:26:23 PM: Starting to prepare the repo for build
12:26:23 PM: Preparing Git Reference refs/heads/main
12:26:24 PM: Failed during stage "preparing repo": From https://github.com/zwyyy456/hugo_papermod_blog
 * branch            main       -> FETCH_HEAD
   1a77de2..dda4b64  main       -> origin/main
warning: ae66501cc9d0ba080c7fcb5a852c0f9fe8649d0d:.gitmodules, multiple configurations found for "submodule.themes/PaperMod.path". Skipping second one!
warning: ae66501cc9d0ba080c7fcb5a852c0f9fe8649d0d:.gitmodules, multiple configurations found for "submodule.themes/PaperMod.url". Skipping second one!
Fetching submodule themes/PaperMod
From https://github.com/adityatelange/hugo-PaperMod
   0dfff4e..8411030  exampleSite -> origin/exampleSite
   b288ede..0989c28  master      -> origin/master
fatal: remote error: upload-pack: not our ref ea64dacc943c5a0d3163dd75dcc11d5e791a37b4
Errors during submodule fetch:
	themes/PaperMod
: exit status 1
```

## 解决方案

问题出在访问 `https://github.com/adityatelange/hugo-PaperMod/tree/ea64dacc943c5a0d3163dd75dcc11d5e791a37b4` 的结果已经是 404 了，大概就是 PaperMod 主题提交记录改变之类的原因导致的，而 Netlify 还在使用这个 sha 记录，解决方案也很简单，点击 `retry -> Clear cache and deploy site` 即可。

> 对于 Git 的细节我并不清楚，这里的问题原因看看就好。

## 参考

[Fatal: remote error: upload-pack: not our ref](https://answers.netlify.com/t/fatal-remote-error-upload-pack-not-our-ref/101335)