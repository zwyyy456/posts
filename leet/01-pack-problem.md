---
title: "01 kanpsack problem"
date: 2022-10-01T15:08:23+08:00
lastmod: 2022-10-01T15:08:23+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "dynamic programming"]
description: "notes and thoughts about 01-pack-problem" #描述
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
[01-pack-problem](https://www.acwing.com/problem/content/2/)

There are $N$ items and a pack with capacity of $V$, and each item can only be used once.

The volume of the $i$-th item is $v_i$, and vaule is $w_i$. Please solve which items can be put into the pack so that the value is the greatest and the total volume of these items dosen't exceed the capacity of the pack.

## Solution
It's a classic problem of dynamic programming. First, let's consider the sense of `dp[i][j]`, which `i` means we only consider the first `i` items(`i`: $1\sim N$).

So, `dp[i][j]` denotes the greatest value of items in pack in circumstance that total volume of items in pack is $j$ and only the first `i` items are considered(There may not be `i` items in pack).

We can consider the recursive relation of `dp[i][j]` in two cases.
- When `i`-th item is not in pack: `dp[i][j] = dp[i - 1][j]`
    - In this case, only first `i - 1` items are considered and total volume is still `j`.
- When `i`-th item is in pack, `dp[i][j] = dp[i - 1][j - v[i]] + w[i]`
    - In this case, volume of the first `i - 1` items considered is `j - v[i]`.

## Code
```cpp
#include <algorithm>
#include <iostream>
#include <string>
using namespace std;
const int N = 1010; // volume is less than 1000， number of items is small than 1000.

int main() {
    int n, m; // n denotes number of items，m denotes capacity of pack
    cin >> n >> m;
    int dp[N][N] = {0};
    int v[N] = {0}; // volume
    int w[N] = {0}; // value
    for (int i = 1; i <=n; i++)
        cin >> v[i] >> w[i];
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            dp[i][j] = dp[i - 1][j];
            if (j >= v[i]) // Total volume can't be less than v[i]
                dp[i][j] = max(dp[i][j], dp[i - 1][j - v[i]] + w[i]);
        }
    }
    // int res = 0;
    // for (int j = 1; j <=m; j++) {
    //     res = max(res, dp[n][j]); // Traversal is not needed, just dp[n][m]
    // }
    cout << dp[n][m] << endl;
    return 0;
}
```

## Optimization
If we analyze the code above, we can find only `dp[i - 1][j]` is needed to caculate `dp[i][j]`, `dp[i - 2][j], dp[i - 3][j]` is not needed. So one dimension is enough for `dp` array, which the total volume is index of array.

So, what if it's directly written as:
```cpp
for (int i = 1; i < n; i++) {
    for (int j = 1; j <= m; j++)
        if (j >= v[i])
            // max actually:
            // max(dp[i][j], dp[i][j - v[i]] + w[i])
            dp[j] = max(dp[j], dp[j - v[i]] + w[i]);  
}
```

The answer is NO! The reason is showed in the comment in the code above. It should be `max(dp[i - 1][j], dp[i - 1][j - v[i]] + w[i])`.

The correct way to write it should be:
```cpp
for (int i = 1; i < n; i++) {
    for (int j = m; j >= v[i]; j--)
        // Because dp[j], dp[j - v[i]] was assigned value in last outer `i` loop，
        // so it's the same with dp[i - 1][j - v[i]].
        dp[j] = max(dp[j], dp[j - v[i]] + w[i]) 
}
```

If `dp[0] = 0, dp[i] = -INF`，so the state can only be transformed from `dp[0]`, which can solve for the case that the total volume is exactly $V$.

The whole code after optimization:
```cpp
#include <algorithm>
#include <iostream>
#include <string>
using namespace std;
const int N = 1010; // volume is less than 1000， number of items is small than 1000.

int main() {
    int n, m; // n denotes number of items，m denotes capacity of pack
    cin >> n >> m;
    int dp[N][N] = {0};
    int v[N] = {0}; // volume
    int w[N] = {0}; // value
    for (int i = 1; i <=n; i++)
        cin >> v[i] >> w[i];
    for (int i = 1; i <= n; i++) {
        for (int j = m; j >= v[i]; j--) {
                // Total volume can't be less than v[i]
                dp[j] = max(dp[j], dp[j - v[i]] + w[i]);
        }
    }
    // int res = 0;
    // for (int j = 1; j <=m; j++) {
    //     res = max(res, dp[n][j]); // Traversal is not needed, just dp[n][m]
    // }
    cout << dp[m] << endl;
    return 0;
}
```

## Examples
- [416.partition-equal-subset-sum](https://leetcode.com/problems/partition-equal-subset-sum/)
    - [416.partition-equal-subset-sum-solution](https://zwyyy456.vercel.app/posts/tech/416.partition-equal-subset-sum)
- [1049.last-stone-weight-ii](https://leetcode.com/problems/last-stone-weight-ii/)
    - [1049.last-stone-weight-ii-solution](https://zwyyy456.vercel.app/posts/tech/1049.last-stone-weight-ii/)
- [494.target-sum](https://leetcode.com/problems/target-sum/)
    - [494.target-sum-solution](https://zwyyy456.vercel.app/posts/tech/494.target-sum/)
- [474.ones-and-zeroes](https://leetcode.com/problems/ones-and-zeroes/)
    - [474.ones-and-zeroes-solution](https://zwyyy456.vercel.app/posts/tech/474.ones-and-zeroes/)