---
title: "一维动态规划 - 基础版"
date: 2022-09-28T19:23:30+08:00
lastmod: 2022-09-28T19:23:30+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "dynamic programming"]
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
一般来说，一维动态规划的问题，其输入的参数一般是$n$，而所求结果有点像数列$a_n$，或者说$f(n)$($f$可以认为是函数或者说对应关系)，同时$a_n$与之前的$a_{n-1},a_{n-2},...a_{1}$有一个确定的对应的关系，例如$a_n = a_{n-1} + a_{n-2}$(斐波那契数列)

## 解题步骤
数列即可与编程语言中的数组对应起来，在找到数列之间的迭代关系时，即可编写`for`循环来求解。

## 例题
- [509.斐波那契数](https://leetcode.cn/problems/fibonacci-number/)
    - [509.斐波那契数 - 题解](https://blog.zwyyy456.tech/zh/posts/tech/509.fibonacci-number/)
- [70.爬楼梯](https://leetcode.com/problems/climbing-stairs/)
    - [70.爬楼梯 - 题解](https://blog.zwyyy456.tech/zh/posts/tech/70.climbing-stairs/)
- [746.使用最小花费爬楼梯](https://leetcode.cn/problems/min-cost-climbing-stairs/)
    - [746.使用最小花费爬楼梯 - 题解](https://blog.zwyyy456.tech/zh/posts/tech/746.min-cost-climbing-stairs/)
- [343.整数拆分](https://leetcode.cn/problems/integer-break/)
    - [343.整数拆分 - 题解](https://blog.zwyyy456.tech/zh/posts/tech/343.integer-break/)
- [62.不同路径](https://leetcode.cn/problems/unique-paths/)
    - [62.不同路径 - 题解](https://blog.zwyyy456.tech/zh/posts/tech/62.unique-paths/)
- [63.不同路径 II](https://leetcode.cn/problems/unique-paths-ii/)
    - [63.不同路径 II-题解](https://blog.zwyyy456.tech/zh/posts/tech/63.unique-paths-ii/)

