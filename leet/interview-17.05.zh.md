---
title: "面试题 17.05.  字母与数字 (Medium)"
date: 2023-03-11T15:31:57+08:00
lastmod: 2023-03-11T15:31:57+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["leetcode"]
tags: ["daily", "hash table", "prefix sum"]
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
[面试题 17.05.  字母与数字 (Medium)](https://leetcode.cn/problems/find-longest-subarray-lcci/)

给定一个放有字母和数字的数组，找到最长的子数组，且包含的字母和数字的个数相同。

返回该子数组，若存在多个最长子数组，返回左端点下标值最小的子数组。若不存在这样的数组，返回一个空数组。

**示例 1:**

```
输入:
["A","1","B","C","D","2","3","4","E","5","F","G","6","7","H","I","J","K","L","M"]

输出:
["A","1","B","C","D","2","3","4","E","5","F","G","6","7"]

```

**示例 2:**

```
输入: ["A","A"]

输出: []

```

**提示：**

- `array.length <= 100000`

## 解题思路
首先使用一个前缀和数组`prefix`，prefix[i]表示前i个数里，数字的数量减去字母的数量，遍历`array`，更新`prefix`，同时在哈希表中查找`key`->`prefix[i]`是否存在：
- 如果存在，比较记录的最长长度`len`，如果大于`len`，则更新`idx = ump[prefix[i]]`，并更新`len = i - ump[prefix[i]]`；
- 否则，更新哈希表，即`ump[prefix[i]] = i`；

## 代码
```cpp
class Solution {
  public:
    vector<string> findLongestSubarray(vector<string> &array) {
        int n = array.size();
        vector<string> res;
        if (n < 2) {
            return res;
        }
        unordered_set<string> ust;
        for (char c = 'a'; c <= 'z'; c++) { // 统计所有的字母
            string s(1, c);
            ust.insert(s);
        }
        for (char c = 'A'; c <= 'Z'; c++) {
            string s(1, c);
            ust.insert(s);
        }
        unordered_map<int, int> ump;
        ump[0] = 0;
        int len = 0;
        int idx = 0;
        vector<int> prefix(n + 1); // prefix[i]表示前i个数里，数字的数量减去字母的数量
        for (int i = 1; i <= n; ++i) {
            if (ust.find(array[i - 1]) == ust.end()) {
                prefix[i] = prefix[i - 1] + 1;
            } else {
                prefix[i] = prefix[i - 1] - 1;
            }
            if (ump.find(prefix[i]) != ump.end()) { // 说明存在以i - 1为右端点的子数组
                if (len < i - ump[prefix[i]]) {
                    len = i - ump[prefix[i]];
                    idx = ump[prefix[i]];
                }
            } else {
                ump[prefix[i]] = i;
            }
        }
        res.resize(len);
        for (int i = idx; i < idx + len; ++i) {
            res[i - idx] = array[i];
        }
        return res;
    }
};
```