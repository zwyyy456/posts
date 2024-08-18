---
title: "质因数分解"
date: 2023-03-06T14:32:15+08:00
lastmod: 2023-03-06T14:32:15+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["math"]
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
## 朴素算法
从$[2, \sqrt(N)]$进行遍历
```cpp
vector<int> GetFactor(int N) {
    vector<int> res;
    for (int i = 2; i * i <= N; ++i) {
        if (N % i == 0) {
            while (N % i == 0) {
                N /= i;
            }
            res.push_back(i);
        }
    }
    if (N != 1) {
        res.push_back(N);
    }
    return res;
}
```

## 朴素算法的证明
首先证明元素均为 $N$ 的素因数：因为当且仅当 `N % i == 0` 满足时，`result` 发生变化：储存 $i$，说明此时 $i$ 能整除 $\frac{N}{A}$，说明了存在一个数 $p$ 使得 $pi=\frac{N}{A}$，即 $piA = N$（其中，$A$ 为 $N$ 自身发生变化后遇到 $i$ 时所除的数。我们注意到 `result` 若在 push $i$ 之前就已经有数了，为 $R_1,\,R_2,\,\ldots,\,R_n$，那么有 `N` $=\frac{N}{R_1^{q_1}\cdot R_2^{q_2}\cdot \cdots \cdot R_n^{q_n}}$，被除的乘积即为 $A$）。所以 $i$ 为 $N$ 的因子。

其次证明 `result` 中均为素数。我们假设存在一个在 `result` 中的合数 $K$，并根据整数基本定理，分解为一个素数序列 $K = K_1^{e_1}\cdot K_2^{e_2}\cdot\cdots\cdot  K_3^{e_3}$，而因为 $K_1 < K$，所以它一定会在 $K$ 之前被遍历到，并令 `while(N % k1 == 0) N /= k1`，即让 `N` 没有了素因子 $K_1$，故遍历到 $K$ 时，`N` 和 $K$ 已经没有了整除关系了。

## 参考
[1. OI-WIKI 分解质因数](https://oi-wiki.org/math/number-theory/pollard-rho/#pollard-rho-算法)

