---
title: "二分答案"
date: 2023-06-01T15:53:13+08:00
lastmod: 2023-06-01T15:53:13+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["binary search", "data structure and algorithms"]
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
二分答案即利用二分查找来得到答案，一般情况下，左边界 $left$ 是 $0$ 或者 $1$；右边界 $right$ 则视题目条件而定，取一个很大的数，然后利用二分查找的思想，来找到答案。

## 二分答案的要求
如果题目能够使用二分答案的思想来解决，那么 $[left, right]$ 范围内，要满足二段性，即对 $[left, res]$ 满足条件 $A$，而 $(res, right]$ 不满足条件 $A$，并且 `res` 的取值范围是连续的。

## 适用情况
如果题目要求满足 xxx 条件下的最大值或者最小值，就可以考虑二分答案，特别的，如果题目要求**最小化的最大值**或者**最大化的最小值**，那么要首先考虑使用二分答案。

## 例题
[2517. 礼盒的最大甜蜜度 (Medium)](https://leetcode.cn/problems/maximum-tastiness-of-candy-basket/)

```cpp
class Solution {
  public:
    int Bsearch(int target, vector<int> &price, int left) {
        int right = price.size();
        while (left < right) {
            int mid = left + (right - left) / 2;
            if (price[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        return left;
    }
    bool Check(int mid, vector<int> &price, int k, int n) {
        int start = 0;
        for (int i = 0; i < k - 1; ++i) {
            start = Bsearch(price[start] + mid, price, start);
            // cout << start << " start\n";
            if (start >= n) { 
                return false;
            }
        }
        return true;
    }
    int maximumTastiness(vector<int> &price, int k) {
        // 先排序，然后考虑是二分答案还是双指针
        sort(price.begin(), price.end());
        // 二分答案，时间复杂度为 log(price[i]) * k * log(n)
        int n = price.size();
        int left = 0, right = (price[n - 1] - price[0]) / (k - 1) + 1; // 先看看 k 行不行，不行就改成 2
        while (left < right) {
            // 左闭右开
            int mid = left + (right - left) / 2;
            if (Check(mid, price, k, n)) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        return left - 1;
    }
};
```

