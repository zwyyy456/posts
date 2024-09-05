---
title: "01 背包问题"
date: 2022-10-01T15:08:30+08:00
lastmod: 2022-10-01T15:08:30+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["data structure and algorithms", "dynamic programming"]
description: "01 背包问题的笔记与思考" #描述
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
## 问题描述
[01 背包问题](https://www.acwing.com/problem/content/2/)
有$N$件物品和一个容量是$V$的背包，每件物品只能使用一次。

第$i$件物品的体积是$v_i$，价值是$w_i$，求解将哪些物品装入背包，可使这些物品总体积不超过背包容量，并且总价值最大。

## 解题思路
动态规划的经典例题，首先考虑`dp[i][j]`的含义，这里`i`表示只考虑前`i`个物品 (`i`从$1\sim N$)，`dp[i][j]`表示总体积为`j`的情况下，考虑前`i`个物品时，背包里的物品的最大价值。

可以分成两种情况考虑`dp[i][j]`的递推关系：
- 第`i`个物品不在背包中时，`dp[i][j] = dp[i - 1][j]`
    - 此时只有前`i - 1`个物品，背包中物品体积仍为`j`。
- 第`i`个物品在背包中时，`dp[i][j] = dp[i - 1][j - v[i]] + w[i]`
    - 前`i - 1`个物品的体积为`j - v[i]`。

初始化，显然`dp[0][0] = 0`。

根据递推关系和初始化条件写`for`循环遍历即可。

## 代码
```cpp
#include <algorithm>
#include <iostream>
#include <string>
using namespace std;
const int N = 1010; // 体积不超过 1000，物品件数也不超过 1000

int main() {
    int n, m; // n 为物品数量，m 为背包体积
    cin >> n >> m;
    int dp[N][N] = {0};
    int v[N] = {0}; // 体积
    int w[N] = {0}; // 价值
    for (int i = 1; i <=n; i++)
        cin >> v[i] >> w[i];
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            dp[i][j] = dp[i - 1][j];
            if (j >= v[i]) // 当前总体积肯定不能小于 v[i]，如果小于的话，第 i 个物品不能放
                dp[i][j] = max(dp[i][j], dp[i - 1][j - v[i]] + w[i]);
        }
    }
    // int res = 0;
    // for (int j = 1; j <=m; j++) {
    //     res = max(res, dp[n][j]); // 不需要遍历，直接输出 dp[n][m] 即可
    // }
    cout << dp[n][m] << endl;
    return 0;
}
```

## 优化
分析上面的代码，实际上`dp[i][j]`递推时只会用到`dp[i - 1][j]`，而不会用到`dp[i - 2][j], dp[i - 3][j]`等，因此`dp`数组实际上只需要一维即可，索引为当前总体积。

那么，是否可以直接写成
```cpp
for (int i = 1; i < n; i++) {
    for (int j = 1; j <= m; j++)
        if (j >= v[i])
            // 这里的 max 实际上是
            // max(dp[i][j], dp[i][j - v[i]] + w[i])
            dp[j] = max(dp[j], dp[j - v[i]] + w[i]);  
}
```

不可以！原因已在上面的代码里的注释中给出，应该是`max(dp[i - 1][j], dp[i - 1][j - v[i]] + w[i])`才对。

正确的写法应该是：
```cpp
for (int i = 1; i <= n; i++) {
    for (int j = m; j >= v[i]; j--)
        // 因为 dp[j], dp[j - v[i]] 只在上一次的 i 循环中才被赋值了，
        // 所以这里用的实际上是 dp[i - 1][j - v[i]]
        dp[j] = max(dp[j], dp[j - v[i]] + w[i]); 
}
```

如果`dp[0] = 0， dp[i] = -INF`，那么状态只能从`dp[0]`转移过来，可以求解总体积恰为$V$的情况。

优化后的完整代码：
```cpp
#include <algorithm>
#include <iostream>
#include <string>
using namespace std;
const int N = 1010; // 体积不超过 1000，物品件数也不超过 1000

int main() {
    int n, m; // n 为物品数量，m 为背包体积
    cin >> n >> m;
    int dp[N] = {0};
    int v[N] = {0}; // 体积
    int w[N] = {0}; // 价值
    for (int i = 1; i <=n; i++)
        cin >> v[i] >> w[i];
    for (int i = 1; i <= n; i++) {
        for (int j = m; j >= v[i]; j--) {
                // 当前总体积肯定不能小于 v[i]，如果小于的话，第 i 个物品不能放
                dp[j] = max(dp[j], dp[j - v[i]] + w[i]);
        }
    }
    // int res = 0;
    // for (int j = 1; j <=m; j++) {
    //     res = max(res, dp[n][j]); // 不需要遍历，直接输出 dp[n][m] 即可
    // }
    cout << dp[m] << endl;
    return 0;
}
```

## 例题
- [416.分割等和子集](https://leetcode.cn/problems/partition-equal-subset-sum/)
    - [416.分割等和子集 - 题解](https://zwyyy456.vercel.app/zh/posts/tech/416.partition-equal-subset-sum)
- [1049.最后一块石头的重量 II](https://leetcode.cn/problems/last-stone-weight-ii/)
    - [1049.最后一块石头的重量 II-题解](https://zwyyy456.vercel.app/zh/posts/tech/1049.last-stone-weight-ii/)
- [494.目标和](https://leetcode.cn/problems/target-sum/)
    - [494.目标和 - 题解](https://zwyyy456.vercel.app/zh/posts/tech/494.target-sum/)
- [474.一和零](https://leetcode.cn/problems/ones-and-zeroes/)
    - [474.一和零 - 题解](https://zwyyy456.vercel.app/zh/posts/tech/474.ones-and-zeroes/)
