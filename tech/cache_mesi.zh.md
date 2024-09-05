---
title: "CPU 缓存一致性：MESI"
date: 2023-06-07T10:36:47+08:00
lastmod: 2023-06-07T10:36:47+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["csapp"]
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
## 概述

MESI（也称伊利诺斯协议）是一种广泛使用的支持 write-back 策略的缓存一致性协议。

## MESI 状态

我们假设 CPU 中共有 $k$ 个核；
CPU 中每个 cacheline 使用 $4$ 种状态进行标记：

| 状态 | 介绍 | 所有核中该状态的个数
|:-: | - | :-: |
| MODIFIED | 实际上是 exclusive dirty，说明该核的缓存数据被修改了，且并未写入到更低一层存储中；当某个核的缓存处于该状态时，其余核的对应 cacheline 均为 INVALID | 1 |
| EXCLUSIVE | 实际上是 exclusive clean，说明该核的缓存刚从更低一层存储中读取到了最新的数据；当某个核的缓存处于该状态时，其余核的对应 cacheline 均为 INVALID | 1 |
| SHARED | 实际上是 shared clean，说明多个核的缓存从更低一层存储中读到的数据都是最新的，处于该状态的核的数量一定 $\geq 2$，其余核的对应 cacheline 均为 INVALID | $\geq 2$ |
| INVALID | 说明该核的 cacheline 无效 | |

## 状态切换

我们以四个核为例，用四元组如 $(M, I, I, I)$ 表示四个核的 cacheline 状态，假设**只操作第一个核**的 cacheline：

### 执行读操作

- $(M,I,I,I)$ 执行读，状态仍为 $(M, I, I, I)$；
- $(E,I,I,I)$ 执行读，状态仍为 $(E, I, I, I)$；
- $(S,S,I,I)$ 执行读，状态仍为 $(S, S, I, I)$；
- $(I,M,I,I)$ 执行读，将第二个核的 cacheline 的数值 $val$ 写入内存，然后更新第一个核的 cacheline 的值为 $val$，状态切换为 $(S,S,I,I)$；
- $(I,E,I,I)$ 执行读，更新第一个核的 cacheline 为第二个核的数值 $val$，状态切换为 $(S,S,I,I)$；
- $(I,S,S,I)$ 执行读，更新第一个核的 cacheline 为第二个或第三个核的数值 $val$，状态切换为 $(S,S,S,I)$；
- $(I,I,I,I)$ 执行读，从内存中读取对应的地址的数值 $*addr$ 到 cacheline，状态切换为 $(E, I, I, I)$；

### 执行写操作

- $(M,I,I,I)$ 执行写，状态仍为 $(M, I, I, I)$；
- $(E,I,I,I)$ 执行写，状态变为 $(M, I, I, I)$；
- $(S,S,I,I)$ 执行写，将其他所有状态为 $S$ 的 cacheline 的状态全都设置为 $I$，状态变为 $(M, I, I, I)$；
- $(I,M,I,I)$ 执行写，将第二个核的 cacheline 的数值 $val$ 写入内存（为了防止 ABA 问题，这里为什么要写回 TODO(zwyyy)，是否和指令原子性有关），状态变为 $I$，然后更新第一个核的 cacheline 的值为 $val$，再更新值为待写入的值 $write$_$value$，状态切换为 $(M,I,I,I)$；
- $(I,E,I,I)$ 执行写，更新第一个核的 cacheline 为第二个核的数值 $val$，将第二个核的状态设为 $I$，状态切换为 $(M,I,I,I)$；
- $(I,I,I,I)$ 执行写，将 $write$_$val$ 写到 cacheline 中，状态切换为 $(M, I, I, I)$；

### 执行 evict 操作




