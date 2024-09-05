---
title: "数位 DP"
date: 2023-06-06T10:37:04+08:00
lastmod: 2023-06-06T10:37:04+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["dynamic programming", "data structure and algorithms"]
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
## 引入
数位是指把一个数字按照个、十、百、千、万等等一位一位地拆开，关注它每一位上的数字。如果拆的是十进制数，那么每一位数字都是 $0\sim 9$。

数位 DP 一般是用来解决一类特定问题，以 [1012. Numbers With Repeated Digits (Hard)](https://leetcode.com/problems/numbers-with-repeated-digits/) 为例，这一类问题的特征非常明显
1. 要求统计满足一定条件的数的数量；
2. 这些条件经过转化后可以使用“数位”的思想去理解和判断；
3. 输入会提供一个数字区间（有时候只提供上界）来作为统计的限制；
4. 上界很大（例如 $10^{22}$），暴力枚举会超时。

## 思路
对 [1012. Numbers With Repeated Digits (Hard)](https://leetcode.com/problems/numbers-with-repeated-digits/)，思路如下：

首先，正难则反，我们可以考虑 $[1, n]$ 范围内无重复数字的正整数的个数，记为 $res$，然后最终结果就是 $n - res + 1$（至于为什么还要再加上 $1$，后面会说明）。

因此问题转化为求 $[1, n]$ 范围内无重复数字的正整数的个数，符合上述的**数位 DP** 的特征，我们可以从记忆化搜索的角度去考虑，首先将 $n$ 转化为对应的字符串，对字符串的每一位，枚举每一位可能的数，如果最后组成的数字满足 $num < n$，那么 $res += 1$，这里很容易想到 `dfs(string &str, int idx, int mask)`，$mask$ 以二进制的形式表示 $0\sim 9$ 范围内的数是否被选择过；

但是仅仅是这样，我们不能方便的判断当前组成的数字是否满足 $num < n$，因此，我们需要一个额外的参数 $is\_limit$。例如对数字 $n = 12345$，如果前面已经选择的数字为 $123$，那么对本次枚举，$is\_limit$ 为 $true$，即数字只能选择 $0\sim4$，又因为 $1,\ 2,\ 3$ 已经选择了，$mask = 14$，因此只有 $0,\ 4$ 可以选，事实上，我们可以发现，在递归的过程中，当且仅当当前 $is\_limit$ 为 $true$，且当前选择的数字与对应数位上的数字相等时，更深一层递归的 $is\_limit$ 仍为 $true$，至此，递归函数为 `dfs(string &str, int idx, int mask, bool is_limit)`；

此外，由于我们递归的终止条件应该是 $idx \geq str.size()$，这样就只能统计出位数与 $n$ 相同的情况，而位数小于 $n$ 的就被忽略了，这是我们不能接受的，因此我们还需要一个参数 $filled$ 来标记之前的位有没有被填充，假设 $filled$ 为 $false$，说明之前没有被填充，那么有两种选择
- 当前位也不填充，`res += dfs(str, idx + 1, false, mask, filled, cach)`；
- 如果要填充，那么必须从 $1$ 开始填充（因为不能有前导 $0$）,`res += dfs(str, idx + 1, false, mask, true, cach)`。

## 模板
根据以上分析，我们可以总结出一个**数位 DP** 适用的模板，如下：
```cpp
int dfs(string &str, int idx, bool is_limit, int mask, bool filled, vector<vector<vector<vector<int>>>> &cach) {
    if (idx == str.size()) {
        return 1; // 说明找到了一个合理的选择
    }
    if (cach[idx][is_limit][mask][filled] != -1) {
        return cach[idx][is_limit][mask][filled];
    }
    int limit = is_limit ? str[idx] - '0' : 9;
    int res = 0;
    if (!filled) {
        // 前面的数位都没有填充
        res += dfs(str, idx + 1, false, mask, false, cach);
    }
    int left = filled ? 0 : 1;
    for (int i = left; i <= limit; ++i) {
        if (((1 << i) & mask) != 0) {
            // 说明 i 已经选择过了
            continue;
        }
        res += dfs(str, idx + 1, is_limit && (i == str[idx] - '0'), mask | (1 << i), true, cach);
    }
    cach[idx][is_limit][mask][filled] = res;
    return res;
}
```

在这个模板中，由于所有位都不填，也会被统计进结果中，因此 $res$ 比实际的 $res$ 多了 $1$，我们可以在最后计算的时候减去 $1$，也可以把边界条件的返回值修改为如下：
```cpp
if (idx == str.size()) {
    return is_num; // is_num 为 true 说明找到了一个合法的数字，为 false 说明所有位都不填
}
```

## 参考
[OI-Wiki：数位 DP](https://oi-wiki.org/dp/number/)

[0x3f：数位 DP 通用模板](https://leetcode.cn/problems/numbers-with-repeated-digits/solutions/1748539/by-endlesscheng-c5vg/)
