---
title: "位运算与集合"
date: 2023-01-05T14:50:46+08:00
lastmod: 2023-01-05T14:50:46+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms"]
description: "" #描述
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
## 前言

在刷 LeetCode 的时候，我们常常碰到需要枚举同时选择几个元素，或者说枚举选择一个集合的情况，即同时选择 $\lbrace0, 1, 2\rbrace$ 或者 $\lbrace0, 1,3\rbrace$ 等，这里集合中的数字表示要选择的元素的索引。

通常情况下，我们往往会使用哈希表来表示集合，好处在于可以方便的在 $O(1)$ 时间内确定元素是否处于集合中，坏处则是当我们需要做集合之间的运算，例如求交集或者并集，那么就需要 $O(n)$ 时间才能实现；另一个缺陷就是，当递归函数的可变实参中存在哈希表（或者对哈希表的引用）时，无法通过添加 $cach$ 数组实现记忆化搜索。

于是，我们需要想一个新的办法来表示集合，由于集合可以由全集（包含所有元素的集合）中每个元素的选或者不选来表示，因此，很容易联想到二进制上每一位的 $0$ 和 $1$，例如 $101 = 5$ 表示集合中只有第 $0$ 个元素和第 $2$ 个元素。

使用数学化一点的语言，即集合可以以如下方式压缩成二进制下的一个数字：

$$f(S)=\sum\limits_{i\in S}2^i$$

其中 $i$ 表示集合中的元素在原数组中的索引。$\lbrace a[0], a[1], a[3]\rbrace$ 即可由 $2^0+2^1+2^3 = 13$ 即二进制数 $1101$ 表示。

## 集合与元素

根据上面提到的二进制表示集合的方法，我们可以在 $O(1)$ 的时间内实现集合与元素之间的运算。

具体运算表格参见灵神的 [从集合论到位运算，常见位运算技巧分类总结！](https://leetcode.cn/circle/discuss/CaOJ45/)。
无需记忆，自己做题的时候很容易就能推导出来。

## 集合与集合

集合与集合之间的运算也可以在用二进制数表示集合的情况下，在 $O(1)$ 时间内完成计算。

具体运算表格同样参见灵神的 [从集合论到位运算，常见位运算技巧分类总结！](https://leetcode.cn/circle/discuss/CaOJ45/)。

同样无需记忆，自己做题的时候很容易就能推导出来。

## 遍历集合

在集合用二进制数 $mask$ 表示的情况下，集合中的元素个数可以由 C++ 库函数 `__builtin_popcount(mask)` 计算出来。

设元素范围从 $0$ 到 $n - 1$，挨个判断元素是否在集合 $s$ 中：

```cpp
for (int i = 0; i < n; ++i) {
    if ((s >> i) & 1) { // i 在 s 中，注意 == 运算优先级高于 &
        // 
    }
}
```

## 枚举集合

重头戏来了：设集合为 $s$，从大到小枚举 $s$ 的所有非空子集 $sub$：

```cpp
for (int mask = s; mask != 0; mask = ((mask - 1) & s)) {
    // 处理子集 sub 的逻辑
}
```

暴力的枚举集合的办法是从 $s$ 出发，不断减一直到 $0$，但是这样中途会有很多并不是 $s$ 的子集的情况。

假设集合 $s = 10101$，那么它的子集从大到小依次为：

$$\lbrace 10101, 10100, 10001, 10000, 00101, 00100, 00001\rbrace$$

如果忽略掉 $10101$ 中间的两个 $0$，即忽略第一位和第三位的 $0$（位索引从 $0$ 开始），那么它的子集的数字变化与普通的二进制减法是一样的，即：

$$\lbrace 111, 110, 101, 100, 011, 010, 001\rbrace$$

因此，当我们执行 $(mask - 1)$ & $s$ 时，以 $10100$ 为例，相当于强制跳过了 $10100$ 到 $10001$ 中间那些第一位和第三位数字不为 $0$ 的数。

套用灵神的说法，以 $10100$ 为例，普通的二进制减法会把最低位的 $1$ 变成 $0$，把这个最低位的 $1$ 右边的 $0$ 都变成 $1$，即 $10100\rightarrow 10011$，我们这个压缩版的二进制减法，也是把最低位的 $1$ 变成 $0$，但对这个最低位的 $1$ 右边的 $0$，并不会全都变成 $1$，而是只保留 $s = 10101$ 中存在的 $1$，其他的会依旧是 $0$。

## Gosper's Hack

Gosper's Hack 算法是生成 $n$ 元集合中所有包含 $k$ 个元素的子集的算法。

这里先给出 Gosper's Hack 算法的代码

```cpp
while (x < uplimit) {
    int lowbit = x & (-x);
    int left = x + lowbit;
    int right = ((x ^ (x + lowbit)) / lowbit) >> 2;
    x = left | right;
}
```

接下来讲一下 Gopser's Hack 算法的思想：

对一个二进制数，例如 $110110$，我们需要找到它从左往右的最后一个 $01$，然后把这个 $01$ 变成 $10$，再把它右边的 $1$ 全部集中到最右边（这里右边的 $1$ 显然都是连续的，否则与最后一个 $01$ 矛盾），即 $110110\rightarrow 111001$。

在举了例子之后，Gosper's Hack 算法的思想其实很好理解。

我们利用 $x + lowbit(x)$ 得到的结果，就是将 $x$ 的第一个 $01$ 变成 $0$，同时右边的数全都变成 $0$，即 $110110\rightarrow 111000$，如果我们使用 $x \oplus (x + lowbit(x))$，即可得到 $x$ 从最后一个 $01$ 起的右边的数，即 $110110\rightarrow 001110$，我们再除以 $lowbit$，即可去掉 $x \oplus (x + lowbit(x))$ 的最右边的连续的 $0$，又因为 $x + lowbit(x)$ 会将这个最后一个的 $01$ 变成 $10$，$01 \oplus 10 = 11$，因此 $(x \oplus(x + low)) / lowbit(x)$ 的 $1$ 的个数比 $x$ 的最后一个 $01$ 的右边的 $1$ 的个数还多了 $2$ 个，于是我们再右移两位，即得到了我们需要 $right$。 

## 参考

[从集合论到位运算，常见位运算技巧分类总结！](https://leetcode.cn/circle/discuss/CaOJ45/)

[算法学习笔记(75): Gosper's Hack](https://zhuanlan.zhihu.com/p/360512296)
