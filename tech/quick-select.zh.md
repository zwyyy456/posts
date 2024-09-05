---
title: "快速选择算法"
date: 2023-04-11T19:55:41+08:00
lastmod: 2023-04-11T19:55:41+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["data structure and algorithms"]
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
## 问题描述
给定一个长度为$n$的数组，如何在$O(n)$的时间复杂度内找到第$k$大的数。

## 思路
朴素的想法是先排序，然后直接找到第$k$个元素，时间复杂度为$O(n\log n)$。

我们可以利用快速排序的思想来解决这个问题，考虑快速排序的划分过程，在快速排序的“划分”结束后，数组$A_p \cdots A_r$被分成了$A_p\cdots A_q$和$A_{q+1}\cdots A_r$，此时可以按照左边元素的个数（$q-p+1$）和$k$的大小关系来判断是只在左边还是右边递归的求解。

## 代码
```cpp
template <Typename T> // 类型T需要定义 < 运算
// arr 为查找范围数组，rk 为需要查找的排名（从 0 开始），len 为数组长度
T find_kth_element(T arr[], int rk, const int len) {
    if (len <= 1) {
        return arr[0];
    }
    // 随机选择基准
    const T pivot = arr[rand() % len];
    // i 当前操作的元素
    // j 第一个等于pivot的元素
    // k 第一个大于pivot的元素
    // 完成一趟三路快排，将序列分为：
    // 小于 pivot 的元素 ｜ 等于 pivot 的元素 ｜ 大于 pivot 的元素
    int i = 0, j = 0, k = len;
    while (i < k) {
        if (arr[i] < pivot) {
            swap(arr[i++], arr[j++]);
        } else if (arr[i] > pivot) {
            swap(arr[i], arr[--k]);
        } else {
            ++i;
        }
    }
    // 根据要找的排名与两条分界线的位置，去不同的区间递归查找第k大的数
    // 如果小于pivot的元素个数比k多，则第k大的元素一定是一个小于pivot的数
    if (rk < j) {
        return find_kth_element(arr, rk, j);
    } else if (rk >= k){
        // 否则，如果小于pivot和等于pivot的元素加起来也没有k多
        // 则第k大的元素一定是一个大于pivot的元素
        return find_kth_element(arr + k, rk - k, len - k);
    } else {
        // 否则，pivot就是第k大的元素
        return pivot;
    }
}
```
## 优化：中位数的中位数
中位数中的中位数（英文：Median of medians），提供了一种确定性的选择划分过程中分界值的方法，从而能够让找第$k$大的数算法在最坏情况下也能实现线性时间复杂度。

该算法的流程如下：

- 将整个序列划分为 $\left \lfloor \dfrac{n}{5} \right \rfloor$组，每组元素数不超过$5$个；
- 寻找每组元素的中位数（因为元素个数较少，可以直接使用**插入排序**等算法）；
- 找出这$\left \lfloor \dfrac{n}{5} \right \rfloor$组元素中位数中的中位数。将该元素作为前述算法中每次划分时的分界值即可。

该优化后，最坏情况下，算法也有$O(n)$的时间复杂度。

## 参考
[OI-Wiki：快速排序](https://oi-wiki.org/basic/quick-sort/)

