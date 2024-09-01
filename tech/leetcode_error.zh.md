---
title: "Leetcode 常见报错的原因分析"
date: 2023-03-01T19:02:35+08:00
lastmod: 2023-03-01T19:02:35+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["leetcode"]
tags: ["attention", "data structure and algorithms"]
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
## 问题 1
### 问题描述
`Line 522: Char 69: runtime error: applying non-zero offset 18446744073709551615 to null pointer (basic_string.h)`

### 报错原因
`string res = 0`

### 报错分析
这里报错的原因是因为使用了`int`整型变量来初始化`string`。

## AddressSanitizer: stack-overflow

栈溢出，通常是由于使用了缺少终止条件的递归调用。
