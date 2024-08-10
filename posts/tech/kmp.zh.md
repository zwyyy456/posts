---
title: "kmp 算法"
date: 2023-03-27T16:45:04+08:00
lastmod: 2023-03-27T16:45:04+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["kmp", "data structure and algorithms"]
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
## 问题描述
**kmp**算法解决的是字符串匹配问题，即:字符串P是否是字符串S的子串？如果是，它出现在s的哪些位置？这里我们称 **S** 为主串，**P** 为模式串。

## 思路
首先是暴力匹配算法（Brute-Force算法），代码如下：
```cpp
void BruteForce(string s, string p) {
    int len_s = s.size(), len_p = p.size();
    for (int i = 0; i <= len_s - len_p; ++i) {
        int flag = true;
        for (int j = 0; j < len_p; ++j) {
            if (s[i + j] != p[j]) {
                flag = false;
                break;
            }
        }
        if (flag) {
            printf("pos = %d\n", i);
        }
    }
}
```

易得时间复杂度的最坏情况是$O(mn)$的，其中$n$为**s**的长度，$m$为**p**的长度。

为了降低字符串比较的复杂度，我们就必须降低比较的趟数，**kmp**算法的主要思想就是尽可能利用残余信息，即字符串如果在 `j = r` 处匹配失败了，那么前`r`个字符一定匹配成功了，暴力算法中，我们每次不匹配的情况下，都让`j = 0`，即模式串又从头开始匹配，这就完全没有利用比较带来的信息。

这里，我们可以找模式串的前缀子串和后缀子串，并得到对前`r`个字符组成的字符串，相等的前缀子串和后缀子串的最大长度，例如 `p = "aabaac"`，那么对前$5$个字符，这个最大长度为2，所以对模式串 p 应该从 `j = 2` 处开始继续匹配，主串的 `i` 保持不变

因此，我们建立一个 `next` 数组，`next[0] = 0`，`next[k - 1]` 表示前$k$个字符组成的字符串中，相等的前缀子串和后缀子串的最大长度，当 `s[i] != p[j]` 时，`j = next[j - 1]`。

下一步就是求 `next` 数组，假设遍历到 `j = x` 时，这个最大长度为 `now` ，即 `next[x - 1] = now`，那么当 `p[x] == p[now]`，`next[x++] = ++now`，如果 `p[x] != p[now]`，那么 `now = next[now - 1]`。


## 代码
快速计算`next`数组的代码:
```cpp
void SetNext(vector<int> &next, string needle) {
    int x = 1, now = 0;
    while (x < needle.size()) {
        if (needle[x] == needle[now]) {
            next[x++] = ++now;
        } else if (now != 0) {
            now = next[now - 1];
        } else {
            next[x] = 0;
            x += 1;
        }
    }
}
```

基于`next`数组进行比较的代码
```cpp
int strStr(string s, string p) {
    int m = p.size(), n = s.size();
    vector<int> next(m);
    SetNext(next, p);
    int i = 0, j = 0;
    while (i < n && j < m) {
        if (s[i] == p[j]) {
            ++i;
            ++j;
        } else {
            if (j > 0) {
                j = next[j - 1];
            } else {
                ++i;
            }
        }
    }
    if (j >= m) {
        return i - m;
    }
    return -1;
}
```

## 参考
[kmp算法-阮行止](https://www.zhihu.com/question/21923021/answer/1032665486)
