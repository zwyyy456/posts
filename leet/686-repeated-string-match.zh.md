---
title: "686.重复叠加字符串匹配 (Medium)"
date: 2023-03-27T21:22:38+08:00
lastmod: 2023-03-27T21:22:38+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["leetcode"]
tags: ["kmp"]
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
## 问题描述
[686. 重复叠加字符串匹配 (Medium)](https://leetcode.cn/problems/repeated-string-match/)

给定两个字符串 `a` 和 `b`，寻找重复叠加字符串 `a` 的最小次数，使得字符串 `b` 成为叠加后的字符串
`a` 的子串，如果不存在则返回 `-1`。

**注意：** 字符串 `"abc"` 重复叠加 0 次是 `""`，重复叠加 1 次是 `"abc"`，重复叠加 2
次是 `"abcabc"`。

**示例 1：**

```
输入：a = "abcd", b = "cdabcdab"
输出：3
解释：a 重复叠加三遍后为 "abcdabcdabcd", 此时 b 是其子串。

```

**示例 2：**

```
输入：a = "a", b = "aa"
输出：2

```

**示例 3：**

```
输入：a = "a", b = "a"
输出：1

```

**示例 4：**

```
输入：a = "abc", b = "wxyz"
输出：-1

```

**提示：**

- `1 <= a.length <= 10⁴`
- `1 <= b.length <= 10⁴`
- `a` 和 `b` 由小写英文字母组成

## 解题思路
对`a`，叠加至少`(b.size() - 1) / a.size()`，最多`(b.size() - 1) / a.size() + 2`次，然后利用`kmp`算法来匹配`b`和新的`a`。

## 代码
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

