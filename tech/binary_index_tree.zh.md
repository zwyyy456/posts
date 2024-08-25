---
title: "树状数组"
date: 2023-06-23T23:55:33+08:00
lastmod: 2023-06-23T23:55:33+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms"]
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
## 引入

数装数组是一种支持**单点修改**和**区间查询**的数据结构。

> 这里的**区间查询**一般指求和。

普通树状数组维护的信息以及运算要**满足结合律并且可以差分**。

## 定义

考虑下标从 $1$ 开始的数组 $a[1]\sim a[8]$，如下图所示：

![w87mfh4LBqotN1W](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070021.png)

我们可以发现：
- $c[2]$ 管辖 $\sum\limits_{i = 1}^2 a[i]$；
- $c[4]$ 管辖 $\sum\limits_{i = 1}^4 a[i]$；
- $c[6]$ 管辖 $\sum\limits_{i = 5}^6 a[i]$；
- $c[8]$ 管辖 $\sum\limits_{i = 1}^8 a[i]$；

即 $c[x]$ 管辖 $\sum\limits_{i = x - (x \text{AND} -x) + 1}^x a[i]$。例如 $6\text{AND}(-6)  + 1 = 5$。

同时我们定义 $\text{lowbit}(x) = x \text{AND}  -x$

## 使用

那么如何使用树状数组呢，第一步是初始化，我们先假定原数组也是下标从 $1$ 开始，从 $0$ 开始那么做相应变换即可。

### 初始化

我们先令 $c[i] = 0$，然后遍历 $a[j]$，每次遍历相当于是（假设原数组元素均为 $0$，然后将其值加上 $a[i]$），即初始化树状数组相当于是做了 $n$ 次单点修改。

```cpp
void build(vector<int> &a, vector<int> &c) {
    for (int i = 1; i < a.size(); ++i) {
        update(i, c, a[i]);
    }
}
```

### 单点修改

每次修改 $i$ 处的值，只会影响到 $i$ 以及 $t = i + \text{lowbit}(i)$，并令 $i = t$，如此循环。

```cpp
int lowbit(int x) {
    return x & (-x);
}
void update(int idx, vector<int> &c, int val) {
    while (idx < c.size()) {
        c[idx] += val;
        idx = idx + lowbit(idx);
    }
}
```

### 区间查询

例如求 $[a, b]$ 之间的和。

```cpp
int getsum(vector<int> &c, int idx) {
    int res = 0;
    while (idx > 0) {
        res += c[idx];
        idx = idx - lowbit(idx);
    }
    return res;
}

int getsum(vector<int> &c, int a, int b) {
    return getsum(c, b) - getsum(a - 1);
}
```