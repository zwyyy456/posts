---
title: "686 Repeated String Match"
date: 2023-03-27T21:22:53+08:00
lastmod: 2023-03-27T21:22:53+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: [""]
tags: [""]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
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
[686. Repeated String Match (Medium)](https://leetcode.com/problems/repeated-string-match/)

Given two strings `a` and `b`, return the minimum number of times you should repeat string  `a` so
that string `b`is a substring of it. If it is impossible for `b`  to be a substring of `a` after
repeating it, return `-1`.

**Notice:** string `"abc"` repeated 0 times is `""`, repeated 1 time is `"abc"` and repeated 2 times
is `"abcabc"`.

**Example 1:**

```
Input: a = "abcd", b = "cdabcdab"
Output: 3
Explanation: We return 3 because by repeating a three times "abcdabcdabcd", b is a substring of it.

```

**Example 2:**

```
Input: a = "a", b = "aa"
Output: 2

```

**Constraints:**

- `1 <= a.length, b.length <= 10⁴`
- `a` and `b` consist of lowercase English letters.

## Solution
kmp, make `s =  x * a`, $x$ is a number.

## Code
```cpp
class Solution {
public:
    bool check(string &s, string b) {
        vector<int> next(b.size());
        int x = 1, now = 0;
        while (x < b.size()) {
            if (b[x] == b[now]) {
                next[x++] = ++now;
            } else if (now == 0) {
                next[x++] = 0;
            } else {
                now = next[now - 1];
            }
        }
        int i = 0, j = 0;
        while (i < s.size() && j < b.size()) {
            if (s[i] == b[j]) {
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
        return j >= b.size();
    }
    int repeatedStringMatch(string a, string b) {
        int left = (b.size() - 1) / a.size();
        string s;
        if (left == 0) {
            s = a;
        } else {
           for (int i = 0; i < left; ++i) {
                s += a; 
           } 
        }
        for (int i = 0; i < 3; ++i) {
            if (check(s, b)) {
                return s.size() / a.size();
            } else {
                s += a;
            }
        }
        return -1;

    }
};
```