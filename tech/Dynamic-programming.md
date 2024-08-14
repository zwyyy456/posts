---
title: "Dynamic Programming"
date: 2022-09-29T16:25:12+08:00
lastmod: 2022-09-29T16:25:12+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "dynamic programming"]
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
## Description
Usually, One-dimensional dynamic planning problem, the parameter is always $n$, the result is similar to number sequence $a_n$, or $f(n)$($f$ can be viewed as function or corresponding relationship). At the same time, there will be certain corresponding relationship between $a_n$ and $a_{n - 1}, a_{n - 2}...a_{1}$, such as $a_n = a_{n-1} + a_{n-2}$(fibonacci sequence).

## Solution
Number sequence can be corresponded with array in programming language such as C++. If you find the recursive relationship among number sequence, you can write traversal code using `for` loop to get the answer.

## Examples
- [509.fibonacci-number](https://leetcode.cn/problems/fibonacci-number/)
    - [509.fibonacci-number-solution](https://blog.zwyyy456.tech/posts/tech/509.fibonacci-number/)
- [70.climbing-stairs](https://leetcode.com/problems/climbing-stairs/)
    - [70.climbing-stairs-solution](https://blog.zwyyy456.tech/posts/tech/70.climbing-stairs/)
- [746.min-cost-climbing-stairs](https://leetcode.cn/problems/min-cost-climbing-stairs/)
    - [746.min-cost-climbing-stairs-solution](https://blog.zwyyy456.tech/posts/tech/746.min-cost-climbing-stairs/)
- [343.integer-break](https://leetcode.cn/problems/integer-break/)
    - [343.integer-break-solution](https://blog.zwyyy456.tech/posts/tech/343.integer-break/)
- [62.unique-paths](https://leetcode.cn/problems/unique-paths/)
    - [62.unique-paths-solution](https://blog.zwyyy456.tech/posts/tech/62.unique-paths/)
- [63.unique-paths-ii](https://leetcode.cn/problems/unique-paths-ii/)
    - [63.unique-paths-ii-solution](https://blog.zwyyy456.tech/posts/tech/63.unique-paths-ii/)


