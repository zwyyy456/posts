---
title: "完全背包问题"
date: 2022-10-03T13:50:44+08:00
lastmod: 2022-10-03T13:50:44+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["data structure and algorithms", "dynamic programming"]
description: "UKP" #描述
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
[完全背包问题](https://www.acwing.com/problem/content/3/)

有$N$件物品和一个容量是$V$的背包，每件物品都有无限件可用。

第$i$种物品的体积是$v_i$，价值是$w_i$。求解将哪些物品装入背包，可使这些物品总体积不超过背包容量，且总价值最大。

## 解题思路
### 内层嵌套循环
[01背包问题](https://blog.zwyyy456.tech/zh/posts/tech/01-pack-problem/) 每样物品只能使用一件，而针对完全背包问题，我们只需要在内层有关体积的循环中，再添加一层循环，枚举一共使用了多少件物品$i$即可。
```cpp
for (int i = 1; i <= n; i++) {
    for (int j = m; j >= v[i]; j--) {
        for (k = 1; k * v[i] <= j; k++) {
            dp[j] = max(dp[j], dp[j - k * v[i] + k * w[i]);
        }
    }
}
```

### 更改遍历方向
在[01背包问题](https://blog.zwyyy456.tech/zh/posts/tech/01-pack-problem/)中，我们内层关于体积的循环，是从大到小的，这是为了保证在比较`max(dp[j], dp[j - v[i]] + w[i])`时，使用的是上一次`i`循环的数值；

而在完全背包问题中，内层关于体积的循环，修改成从小到大即可，此时`dp = max(dp[j], dp[j - v[i]] + w[i])`中，`dp[j - v[i]] + w[i]`使用的就是本次`i`循环中的数值，而`i`循环中,`dp[j - v[i]] = max(dp[j - v[i]], dp[(j - v[i]) - v[i]] + w[i])`,依次往前递推，总能找到那个最大值`dp[j - k * v[i]] + k * w[i]`。

事实上，内层的体积循环中，遍历方向**由大到小**就是保证每个物品只使用**一次**，**由小到大**则是可以使用**无限次**。
```cpp
for (int i = 1; i <= n; i++) {
    for (int j = v[i]; j <= m; j++) {
        dp[j] = max(dp[j], dp[j - v[i]] + w[i]);
    }
}
```

## 例题
- [518.零钱兑换II](https://leetcode.cn/problems/coin-change-2/)
    - [518.零钱兑换II-题解](https://blog.zwyyy456.tech/zh/posts/tech/518.coin-change-ii/)
- [377.组合总和IV](https://leetcode.cn/problems/combination-sum-iv/)
    - [377.组合总和IV-题解](https://blog.zwyyy456.tech/zh/posts/tech/377.combination-sum-iv/)
- [322.零钱兑换](https://leetcode.cn/problems/coin-change/)
    - [322.零钱兑换-题解](https://blog.zwyyy456.tech/zh/posts/tech/322.coin-change/)
