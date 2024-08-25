---
title: "monotone stack"
date: 2022-11-11T15:55:04+08:00
lastmod: 2022-11-11T15:55:04+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "monotone stack"]
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
## Description
### brief
Monotone stack is a stack whose elements(from top to bottom) are (strictly) monotonically increasing or decreasing.

*Monotone increasing stack*: the element which is smaller than the element in the top can be pushed into stack, else we will pop the element in the top, until the stack is empty or the element is smaller than the element in the top, then we push the element into the stack. This data structure is usually used for problems to find first element that is larger than certain element.

*Monotone decreasing stack*: is the opposite of a monotonically increasing stack, used for problems to find first element that is smaller than certain element.

### how to judge
Monotonically increasing/decreasing stacks are generally determined by the order in which they exit the stack

### the sentry tips
Sometimes, if all the elements in the array is needed, that is: all the elements in the stack will be popped. It's very likely that the border will not be passed because it is not considered. So we can use the *sentinel method*.

For example, adding a `-1` at the end of {1, 3, 4, 5, 2, 9, 6} as a sentinel, it becomes {1, 3, 4, 5, 2, 9, 6, -1}, a trick that simplifies the code logic.

## Examples


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
