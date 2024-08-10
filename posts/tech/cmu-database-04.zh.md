---
title: "Storage Models and Compression"
date: 2023-08-24T11:35:31+08:00
lastmod: 2023-08-24T11:35:31+08:00 #更新时间
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
## Database workloads

### On-Line Transaction Processing(OLTP)

大量的更新、读取，但是每次只读取或者写入一小部分数据。

### On-Line Analytical Processing(OLAP)

每次读取大量的数据，一般来说用于统计分析。

#### Decomposition storage model (DSM)

每个 page 存储的是特定 column 的数据，而非一堆 tuple（一堆行数据），

- 这样的好处是，当我们需要读取整个 table 的某几列时，我们不需要读取所有的 page，只需要读取特定的几个 page 即可。

    - 并且这样有利于 query processing 和 data compression。

- 缺点在于，插入、删除、以及查找特定行会变慢。

### Hybird Transaction + Analytical Processing


## Data compression

### Columnar compression

