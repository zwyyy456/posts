---
title: "跳表"
date: 2023-06-06T18:28:54+08:00
lastmod: 2023-06-06T18:28:54+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
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
## 跳表介绍
跳表是一种数据结构，使得包含 $n$ 个元素的有序序列的查找和插入操作的平均时间复杂度都是 $O(\log n)$，与红黑树、AVL 性能类似。

**跳表**的快速查询效果是通过维护一个多层次的**链表**实现的，且与前一层（下面一层）的链表元素相比，每一层链表中的元素的数量更少，一开始，算法在最上层（也是最稀疏的一层）查找，直到要查询的元素在该层相邻的两个元素中间。这时，算法将跳转到下一个层，重复刚才的搜索，直到找到需要查找的元素为止如下图所示：

![IXgAsbN6ewiJQn8](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065954.jpg)

层数索引从低到高逐渐递增。

## 描述
跳表是按照层构建的，跳表的最底层是一个普通的有序链表，每个更高层都是下层列表的子列表，相当于下层列表的快速通道，这里可以类比一下 B树 或者 B+树。

在跳表中，第 $i$ 层的元素会有概率 $p$ 出现在第 $i + 1$ 层中（$p$ 通常取 $\frac{1}{2}$ 或者 $\frac{1}{4}$），每个元素平均出现在 $\frac{1}{1 - p}$ 个列表中。

一般认为，列表有 $log_{\frac{1}{p}}n$ 层，在后面的实现中，我们固定了列表的层数为 $klevel = 8$，动态层数的列表实现起来比较复杂。

在查找目标元素时，从顶层列表、头元素起步。算法沿着每层链表搜索，直至找到一个大于或等于目标的元素，或者到达当前层列表末尾：
- 如果该元素等于目标元素，则表明该元素已被找到；
- 如果该元素大于目标元素或已到达链表末尾，则退回到当前层的上一个元素，然后转入下一层进行搜索。

每层链表中预期的查找步数最多为 $\frac{1}{p}$，而层数为 $\log_{\frac{1}{p}}n$，因此查找的总体步数为 $\frac{\frac{1}{p}}{\log_{\frac{1}{p}}n}$，$p$ 是常数，因此总体查找的时间复杂度为 $O(\log n)$ 的。

跳跃列表不像平衡树等数据结构那样提供对最坏情况的性能保证：由于用来建造跳跃列表采用随机选取元素进入更高层的方法，在小概率情况下会生成一个不平衡的跳跃列表（最坏情况例如最底层仅有一个元素进入了更高层，此时跳跃列表的查找与普通列表一致）。

但是在实际中它通常工作良好，随机化平衡方案也比平衡二叉查找树等数据结构中使用的确定性平衡方案容易实现。

## 实现
以 [1206. Design Skiplist (Hard)](https://leetcode.com/problems/design-skiplist/) 为例，进行**跳表**的简单实现。

![ldwYXMuz928Qesi](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065956.jpg)

由上述图片，我们可以构想出结点的数据结构：
- 结点值 $val$；
- 存储当前结点每一层的 $next\_$ 指针（这里使用 `vector` 存储）。
    - 为了方便理解，我们其实可以把每个结点都看成图中 $klevel$ 高度，只是我们只画出来 $next\_[i]$ 不为 $nullptr$ 的对应层罢了。

```cpp
const int klevel = 8;
struct Node {
    int val_;
    vector<Node *> next_; // next[i] 表示当前结点在第 i 层的 next，i 从 0 开始

    Node(int val) :
        val_(val), next_{klevel, nullptr} { // 初始化
    }
};
```

之后，我们使用一个辅助函数 `void Find(int target, veoctor<Node *> pre);` 来存储每一层中：满足值小于 $target$ 并且值最大的结点。

```cpp
void Find(int target, vector<Node *> &pre) {
    // 辅助函数，找到每一层 pre[i] 小于 target 的最大的 i
    // 从头结点开始遍历每一层
    for (int i = 0; i < klevel; ++i) {
        pre[i] = nullptr;
    }
    auto *ptr = head;
    for (int i = klevel - 1; i >= 0; --i) { // 从上层开始往下层找，这里从下往上是不是也可以？
        // 如果当前层 i 对应结点的 next 不为空，且它的值小于 target，则 ptr 往前走指向这一层 ptr 的 next
        while ((ptr->next_[i] != nullptr) && ptr->next_[i]->val_ < target) {
            ptr = ptr->next_[i];
        }
        pre[i] = ptr;
    }
}
```

有了这个辅助函数之后，我们的后续操作就方便很多了：

### 查找（search）
首先执行 `Find(target, pre)` 得到每一层小于 $target$ 的最大结点，由于第 $0$ 层是完整的有序链表，我们直接在第 $0$ 层查找元素即可：

```cpp
bool search(int target) {
    Find(target, pre_);
    auto *ptr = (pre_[0]->next_)[0];
    return ptr != nullptr && ptr->val_ == target;
}
```

### 插入（add）
首先执行 `Find(target, pre)` 得到每一层小于 $target$ 的最大结点。从第 $0$ 层开始，执行类似链表的插入操作，然后利用 `rand()` 来决定是否执行后续层的插入操作。

```cpp
void add(int num) {
    Find(num, pre_);
    auto *ptr = new Node(num);
    for (int i = 0; i < klevel; ++i) {
        ptr->next_[i] = pre_[i]->next_[i]; // 其实就是链表的插入操作
        pre_[i]->next_[i] = ptr;
        if (random() % 2 == 1) {
            break;
        }
    }
}
```

### 删除
可以先执行查找（search）判断待删除元素是否存在，不存在则返回 $false$，如果存在，则从第 $0$ 层开始，删除该层等于目标值结点，直到不存在目标值的层为止；

```cpp
bool erase(int num) {
    if (!search(num)) {
        return false;
    }
    auto *ptr = pre_[0]->next_[0];
    for (int i = 0; i < klevel && pre_[i]->next_[i] == ptr; ++i) {
        pre_[i]->next_[i] = ptr->next_[i];
    }
    delete ptr;
    return true;
}
```

### 完整实现

```cpp
const int klevel = 8;
struct Node {
    int val_;
    vector<Node *> next_; // next[i] 表示当前结点在第 i 层的 next，i 从 0 开始

    Node(int val) :
        val_(val), next_{klevel, nullptr} {
    }
};
class Skiplist {
  public:
    Skiplist() :
        head(new Node(-1)), pre_(klevel) {
        // 初始化为 -1，表示结点不存在
    }
    void Find(int target, vector<Node *> &pre) {
        // 辅助函数，找到每一层 pre[i] 小于 target 的最大的 i
        // 从头结点开始遍历每一层
        for (int i = 0; i < klevel; ++i) {
            pre[i] = nullptr;
        }
        auto *ptr = head;
        for (int i = klevel - 1; i >= 0; --i) { // 从上层开始往下层找，这里从下往上是不是也可以？
            // 如果当前层 i 对应结点的 next 不为空，且它的值小于 target，则 ptr 往前走指向这一层 ptr 的 next
            while ((ptr->next_[i] != nullptr) && ptr->next_[i]->val_ < target) {
                ptr = ptr->next_[i];
            }
            pre[i] = ptr;
        }
    }
    bool search(int target) {
        Find(target, pre_);
        auto *ptr = (pre_[0]->next_)[0];
        return ptr != nullptr && ptr->val_ == target;
    }

    void add(int num) {
        Find(num, pre_);
        auto *ptr = new Node(num);
        for (int i = 0; i < klevel; ++i) {
            ptr->next_[i] = pre_[i]->next_[i]; // 其实就是链表的插入操作
            pre_[i]->next_[i] = ptr;
            if (random() % 2 == 1) {
                break;
            }
        }
    }

    bool erase(int num) {
        if (!search(num)) {
            return false;
        }
        auto *ptr = pre_[0]->next_[0];
        for (int i = 0; i < klevel && pre_[i]->next_[i] == ptr; ++i) {
            pre_[i]->next_[i] = ptr->next_[i];
        }
        delete ptr;
        return true;
    }

  private:
    Node *head; // 跳表头结点
    vector<Node *> pre_;
};
```

## 参考
[C++ 手写跳表最简单的实现方式](https://leetcode.cn/problems/design-skiplist/solutions/1699167/by-tonngw-ls2k/)

[维基百科：跳跃列表](https://zh.wikipedia.org/wiki/%E8%B7%B3%E8%B7%83%E5%88%97%E8%A1%A8#cite_note-pugh-2)

[OI-Wiki：跳表](https://oi-wiki.org/ds/skiplist/)





