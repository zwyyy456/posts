---
title: "B+ tree"
date: 2023-09-05T14:20:58+08:00
lastmod: 2023-09-05T14:20:58+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cmu", "data base", "B+ tree"]
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
## 概述

数据库中 B Tree 一般都是说的 B+ Tree，严格来说， B+ Tree 是 B Tree 的一种。

B+ Tree 是一种自平衡的树形数据结构，增删查改都能在 $O(\log n)$ 时间内完成。

## 性质

1. B+ Tree 是严格平衡的，叶子结点到根结点的结点数都相同；
2. 除根结点是特殊的之外，其他结点应该处于至少半满的状态：
    - 假设度为 $k + 1$（结点最多有 $k$ 个 key），那么结点应该至少有 $\lfloor k/2\rfloor$ 个 key。
3. B+ tree 的 leaf nodes 包含兄弟结点的指针，即所有的叶子结点以双向链表的形式组织。

> 2-3 树可以看作度为 3 的 B 树。

## B+ Tree nodes

B+ Tree 中，只有叶子结点会存储 value，其他非叶子结点只用于找到对应的叶子结点。

叶子结点的取值在不同数据库中、不同优先级的索引中也不同，通常有两种做法：
1. 存储指向最终 tuple 的指针，例如 page id 和 offset；
2. 直接存储 tuple data；
    - 这一方式不适用于 secondary indexes，second indexes 对应的 value 是主键（primary key）。

叶子结点一般还需要存储相邻 siblings 结点的指针以及一些其他的 metadata，如下图所示（将 key 和 value 分为两个数组存储主要是考虑到 cacheline 的利用）。
 


