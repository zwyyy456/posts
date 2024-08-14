---
title: "并查集"
date: 2023-03-22T17:59:31+08:00
lastmod: 2023-03-22T17:59:31+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["dsu", "data structure and algorithms"]
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
## 引入
并查集是一种用于管理元素所属的集合的数据结构，其实现或者说表现为一片森林，其中，每棵树表示了一个集合，树中的节点表示对应的集合中的元素：

顾名思义，并查集支持两种操作：
- 合并（Union）：合并两个元素所属的集合（合并对应的树）；
- 查询（Find）：查询某个元素所属的集合（即查询对应的树的根节点），这可以用于判断两个元素是否属于同一个集合；

> 并查集在经过修改后还可以支持单个元素的移动、删除；使用动态开点线段树还可以实现可持久化并查集；

## 初始化
初始时，我们设置每个元素都属于一个单独的集合，表示为一棵只有根节点的树，每个根节点的父亲都设置为自己。
```cpp
class Dsu {
    vector<size_t> parent_; // 表示每个节点的父节点
    vector<size_t> size_; // 表示每棵树有多少节点
    Dsu(size_t size) : parent_(size), size_(size, 1) {
        iota(parent_.begin(), parent_.end());
    }
}
```

## 查询
我们只需要沿着树向上移动，直到找到根节点
![Y6kdhQFC54vu3xL](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070002.png)
```cpp
size_t Dsu::find(size_t x) {
    return parent_[x] == x ? x : parent_[x];
}
```

## 查询时进行路径压缩
查询过程中，经过的每个元素都属于该集合，因此我们可以直接将其连接到根节点，以加快后续查询。
![HkderY9M6LQDfzs](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070003.png)
```cpp
size_t Dsu::find(size_t x) {
    return parent_[x] == x ? x : parent_[x] = find(parent_[x]);
}
```

## 合并
要合并两棵树，我们只需要将一棵树的根节点连接到另一棵树的根节点。
![SNWCTj2ryEVitnq](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070005.png)
```cpp
void Dsu::Unite(size_t x, size_t y) {
    parent_(find(x)) = find(y);
}
```

## 启发式合并
即将节点较小或者深度较小的树连接到另一棵，这里以按节点数合并的实现作为参考:
```cpp
void Unite(size_t x, size_t y) {
    x = find(x), y = find(y);
    if (x == y) {
        return ;
    }
    if (size_[x] < size_[y]) {
        swap(x, y);
    }
    parent_[y] = x;
    size_[x] += size_[y];
}
```

## 参考文献
[并查集-OI WIKI](https://oi-wiki.org/ds/dsu/)