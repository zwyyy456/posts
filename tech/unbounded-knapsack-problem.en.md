---
title: "unbounded knapsack problem"
date: 2022-10-04T01:33:48+08:00
lastmod: 2022-10-04T01:33:48+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "dynamic programming"]
description: "UKP" #描述
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
## Description
[Unbounded Knapsack Problem](https://www.acwing.com/problem/content/3/)

There are $N$ kinds of items and a knapsack with the capacity of $V$, each item has **unlimited** pieces available.

The volume of the $i$-th item is $v_i$, and value is $w_i$. Please solve which items can be put into the pack so that the value is the greatest and the total volume of these items dosen't exceed the capacity of the pack.

## Solution
It's a classic problem of dynamic programming and knapsack problem. We can solve the problem in two ways.

### Create a new inner loop
For [01-pack-problem](https://blog.zwyyy456.tech/posts/tech/01-pack-problem/), each item can be used only once, while for UKP, we just need to add another layer of loops to the inner loop about volume, enumerating how many items $i$ are used in total.
```cpp
for (int i = 1; i <= n; i++) {
    for (int j = m; j >= v[i]; j--) {
        for (k = 1; k * v[i] <= j; k++) {
            dp[j] = max(dp[j], dp[j - k * v[i] + k * w[i]);
        }
    }
}
```

### Change traversing direction

For [01-pack-problem](https://blog.zwyyy456.tech/zh/posts/tech/01-pack-problem/), the inner loop about volume, is from large to small. This is to ensure that when `dp[j - v[i] + w[i]` is compared with `dp[j]`, the value used is the same as that in last loop.

While in UKP, the inner loop about volume, we can just change it to "from small to large", then in `dp = max(dp[j], dp[j - v[i] + w[i]])`, the value of `dp[j - v[i]] + w[i]` is that in current loop. While in `i` loop, `dp[j - v[i]] = max(dp[j - v[i]], dp[(j - v[i]) - v[i]] + w[i])`, in descending order, we can always find the maximum value: `dp[j - k * v[i]] + k * w[i]`($k$ could be 0).

Actually, in inner loop about volume, traversing **from large to small** ensures that each item can be used *only once*, while traversing **from small to large** ensures that each item can be used *unlimitedly*.
```cpp
for (int i = 1; i <= n; i++) {
    for (int j = v[i]; j <= m; j++) {
        dp[j] = max(dp[j], dp[j - v[i]] + w[i]);
    }
}
```

## Examples
- [518.coin-change-ii](https://leetcode.com/problems/coin-change-2/)
    - [518.coin-change-ii-solution](https://blog.zwyyy456.tech/posts/tech/518.coin-change-ii/)
- [377.combination-sum-iv](https://leetcode.com/problems/combination-sum-iv/)
    - [377.combination-sum-iv-solution](https://blog.zwyyy456.tech/posts/tech/377.combination-sum-iv/)
- [322.coin-change](https://leetcode.com/problems/coin-change/)
    - [322.coin-change-solution](https://blog.zwyyy456.tech/posts/tech/322.coin-change/)
