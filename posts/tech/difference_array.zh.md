---
title: "差分数组"
date: 2023-10-06T17:50:12+08:00
lastmod: 2023-10-06T17:50:12+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "difference array"]
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
## 介绍

考虑原数组为 $[1, 3, 3, 5, 8]$，我们对相邻元素做差，用 $a_i - a_{i - 1}$，可以得到一个差分数组 $[1, 2, 0, 2, 3]$ $diff$，我们认为 $a_{-1}$ 为 $0$，因此

$$diff[i] = \begin{cases} a[0] & i = 0 \newline a[i] - a[i - 1] & i > 0\end{cases}$$

## 性质

差分数组往往和前缀和数组放在一块讨论，事实上，差分数组的前缀和即可还原出原数组。即：

$$ a[k] = \sum\limits_{i = 0}^k diff[i]$$

差分数组还有一个非常重要的性质，那就是它可以将区间修改（将区间中的每个元素都加一个值或者减一个值）变成单点修改。

例如，加入我们要将区间 $[i, j]$ 中的每个元素都加 $c$，那么我们只需要将 $diff[i]$ 加 $c$，并将 $diff[j + 1]$ 减去 $c$ 即可，由这个差分数组取前缀和还原出来的原数组，就是我们修改后的原数组。



