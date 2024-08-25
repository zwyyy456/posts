---
title: "字符串哈希算法"
date: 2023-05-15T11:52:54+08:00
lastmod: 2023-05-15T11:52:54+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["hash table", "string"]
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
考虑 [1044. 最长重复子串 (Hard)](https://leetcode.cn/problems/longest-duplicate-substring/)，本题思路并不难，可以使用二分答案来解决，假设答案为 `mid`，那么长度大于 `mid` 的子串在 `s` 中只会出现一次，否则至少出现两次。

因此只需要考虑子串在 `s` 中的出现次数即可，比较直接的想法是使用 key 为 `string` 的 `unordered_map`，然而 `unoredere_map` 自带的哈希函数，其时间复杂度和空间复杂度都很高，为 $O(len)$，因此，需要一个简单一点的哈希函数。

## 字符串哈希
参照宫水三叶大佬的 [字符串哈希](https://mp.weixin.qq.com/s?__biz=MzU4NDE3MTEyMA==&mid=2247489813&idx=1&sn=7f3bc18ca390d85b17655f7164d8e660&chksm=fd9cb20acaeb3b1cc78abf05d6fea6d093098998ce877f799ac478247604bd267fbee6fcd989&token=1342991619&lang=zh_CN#rd)。

我们需要使用一个比字符串 `s` 略长的哈希数组 `vector<int> h(s.size() + 10)`，以及次方数组 `vector<int> p(s.size() + 10)`。
对长度为 `len` 的数组，只需要利用前缀和思想 `h[i + len] - h[i] * p[len]` 即可在 $O(1)$ 时间内计算出哈希值。

其中 **`p[0] = 1`，`h[i] = h[i - 1] * P + s[i - 1]`；`p[i] = p[i - 1] * P`**。

$P$ 可以依次取 $131,\ 13131,\ 1313131$ 等，出现哈希碰撞就考虑取更大的质数。

其所使用的哈希函数计算方法本质为：$hash(s) = \sum\limits_{i = 1}^{l} s[i - 1] * P^{l - i}$（其中 $l$ 是字符串 $s$ 的长度）。

对这个前缀和计算公式作一个简单证明：
$hash(s[1...r]) = \sum\limits_{i = 1}^{r}s[i - 1] * P^{r - i}，hash(s[1...l]) = \sum\limits_{i = 1}^{l}s[i - 1]*P^{l - i}$

$hash(s[1...r]) -P^{r - l} * hash(s[1...l])  = \sum\limits_{i = 1}^{l}s[i - 1]*P^{r-l+l-i} +\sum\limits_{i=l+1}^{r}s[i - 1]* P^{r-i}-P^{r-l}*\sum\limits_{i = 1}^{l}s[i - 1]*P^{l - i} = \sum\limits_{i=l+1}^{r}s[i - 1]* P^{r-i}$

而 $hash(s[l + 1...r]) = \sum\limits_{i = l}^{r}s[i - 1] * P^{r - i}$

即 $hash(s[1...r]) - hash(s[1...l]) * P^{r - l} = hash(s[l + 1...r])$

## 代码
```cpp
class Solution {
  public:
    string check(int mid, string &s, vector<uint64_t> &h, vector<uint64_t> &p) {
        unordered_set<uint64_t> substrs;
        for (int i = 0; i + mid <= s.size(); ++i) {
            int j = i + mid;
            long hash = h[j] - h[i] * p[j - i];
            if (substrs.find(hash) != substrs.end()) {
                return s.substr(i, mid);
            }
            substrs.insert(hash);
        }
        return "";
    }
    string longestDupSubstring(string s) {
        // 二分查找答案，左边界为0，右边界为 s.size()，左闭右开
        unordered_map<string, int> substr;
        int left = 0, right = s.size();
        string ans;
        int P = 1313131, n = s.size();
        vector<uint64_t> h(n + 10);
        vector<uint64_t> p(n + 10);
        p[0] = 1;
        for (int i = 0; i < n; ++i) {
            p[i + 1] = p[i] * P;
            h[i + 1] = h[i] * P + s[i];
        }
        while (left < right) {
            int mid = left + (right - left) / 2;
            string tmp_res = check(mid, s, h, p);
            if (!tmp_res.empty()) {
                left = mid + 1;
            } else {
                right = mid;
            }
            ans = ans.size() < tmp_res.size() ? tmp_res : ans;
        }
        return ans;
    }
};
```



