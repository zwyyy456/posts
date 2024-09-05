---
title: "Vscdoe 通过 cookie 登陆美区 LeetCode"
date: 2022-09-27T22:47:15+08:00
lastmod: 2022-09-27T22:47:15+08:00
categories: ["setup"]
tags: ["vscode", "tips"]
description: "" #描述
weight: # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: false # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
---
## 安装插件
vscode 安装 leetcode 插件。
## 使用 cookie 登陆
如果选择使用 github 登陆 leetcode.com，似乎会有无法提交和测试的 bug，而用 cookie 登陆就没有这个问题

### 使用 edge 获取 cookie
使用 Firefox 获取的 cookie 有问题，无法正常登陆
- 右键，选择检查
- 选择网络
- 打开 leetcode 的 problem 页面
![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065715.png)
- 下滑找到 cookie 那一栏，复制 cookie

