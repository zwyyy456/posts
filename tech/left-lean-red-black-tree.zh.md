---
title: "左倾红黑树（LLRB）"
date: 2023-06-24T13:36:26+08:00
lastmod: 2023-06-24T13:36:26+08:00 #更新时间
author: ["zwyyy456"] #作者
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
## 简介

红黑树是平衡二叉查找树的一种，前面我们提到，非平衡的 BST，在只有随机插入和查询的情况下，时间复杂度是 $O(\log n)$ 的，然而，如果同时存在随机插入和随机删除，那么时间复杂度会退化到 $O(\sqrt n)$，这是我们无法接受的。

红黑树在插入和删除时，会维持树的平衡，即保证树的高度在 $[\log n, \log(n + 1)]$，理论上极端情况可能树高最大会到达 $2 * \log n$，实际上很难遇到（尽管这样，还是保证了 $O(\log n)$ 的增删查改时间复杂度。

红黑树其实是 2-3 查找树或者 2-3-4 查找树的一种二叉树的实现方式。其中基于 2-3 树实现的红黑树被称为左倾红黑树（Left-Leaning Red-Black Trees, LLRB），基于 2-3-4 树实现的就是普通的红黑树，左倾红黑树实现起来比红黑树更简单一些（《算法 第四版》里面也主要讲的左倾红黑树），因此先讲左倾红黑树。



## 2-3 查找树

在普通的二叉树中，非叶子结点可以有一个或者两个子结点，如果没有子结点，那么就是叶子结点。

而 2-3 查找树的限制要严格很多。我们将拥有一个 key 和两个链接 的结点称为 2- 结点，拥有两个 key 和三个链接的结点称为 3- 结点。

- 2- 结点含有一个 key，两个 link；
- 3- 结点含有两个不同的 key，三个 link。

> 3- 结点含有两个 key（$key_1 < key_2$），三个链接，左链接指向 2-3 树中的 key 都小于 $key_1$，中间链接指向的 2-3 树中的 key 都满足 $key_1 < key < key_2$，右链接指向的 2-3 树中的 key 都大于 $key_2$。

2-3 查找树**只能**由 2- 结点或者 3- 结点组成（这里不统计空结点（null）），可以看到 2-3 查找树的限制是非常严格的，这也让它可以拥有一些非常理想的性质。

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065916.png)

一颗**完美平衡**的 2-3 查找树中所有空结点（null）到根结点的距离都应该是相同的，后续我们都使用 **2-3 树**指代一颗完美平衡的 2-3 查找树。（其他情况下表示的可能是一种更一般的结构）。

### 查找

将二叉查找树的查找算法一般化就能得到 2-3 树的查找算法。

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065919.png)

### 向 2- 结点中插入新 key

首先进行一次未命中的查找，如果未命中的查找结束于一个 2- 结点，那么我们往这个 2- 结点中插入新的 key，使这个 2- 结点变成 3- 结点就好了，不影响所有结点的树高。

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065920.png)

> 插入 key 时，由于 2-3 树的性质，我们必定是在叶子结点处执行插入。

### 向 3- 结点中插入新的 key

#### 1. 向一个只含有一个 3- 结点的树中插入新 key

先组成一个临时的 4- 结点，然后将 4- 结点中的三个 key 中间的 key 上移作为剩下的 3- 结点的父结点。如下图所示：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065922.png)

注意到，这样操作之后，2-3 树的高度 $+1$ 了，所有原来位置的结点高度均 $+1$。

#### 2. 向一个父结点为 2- 结点的 3- 结点中插入新 key

还是先组成一个临时的 4- 结点，然后将 4- 结点的三个 key 中中间的 key 上移，将这个 key 插入到本来为 2-结点的父结点组成 3- 结点，树的高度不变，如下图所示：


#### 3. 向一个父结点为 3- 结点的 3- 结点中插入新 key

还是先组成一个临时的 4- 结点，然后将 4- 结点的三个 key 中中间的 key 上移，即将这个 key 插入到父结点中：此时情形就可以转化成 $1$ 或者 $2$ 或者 $3$。

如果转换成 $3$，那么就相当于是递归操作。

因此，对于 $3$ 这一插入类型，我们可以从递归的角度去理解它，递归终止条件就是切换到情形 $1$ 或者 $2$，每次递归中执行的操作就是构成一个临时的 4- 结点，然后将 4- 结点的三个 key 中中间的 key 上移，插入到父结点中。

如下图所示：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065924.png)

从插入操作我们可以分析得出，对 2-3 树来说，插入操作不会破坏它的完美平衡！要么整棵树的高度不变，要么整颗树的高度 $+1$，因此，根结点到所有 null 结点的长度依旧是相同的。

2-3 树的树高介于 $\log n$ 和 $\log_3 n$ 之间。

### 2-3 树的删除操作

要学习 LLRB 的删除操作，我们先学习 2-3 树的删除最小的 key 的操作：

如果这个最小的 key 位于树底部的 3- 结点中，那么删除很简单；

但如果要删除的 key 是一个 2- 结点，那么麻烦了，假设我们直接从 2- 结点删除这个 key，那么就剩下了一个 null 结点，这会破坏 2-3 树的完美平衡性，因此，我们需要想办法保证，要删除的最小 key 一定在 3- 结点中，哪怕它原来在 2- 结点中，我们也让它变成 3- 结点。为了保证这一点，我们沿着左 link 向下进行变换，确保当前结点不是 2- 结点（可能是 3- 结点或者临时的 4- 结点）。

首先，根结点可能有两种情况，如果根结点是 2- 结点，并且它两个子结点都是 2- 结点，那么我们需要将这三个结点合并成一个临时的 4- 结点；否则，我们需要保证根结点的左子结点不是 2- 结点，例如如果左子结点是 2- 结点而右子结点是 3- 结点，那么我们可以从右子结点处“借”一个结点给左子结点。

在沿着左链接往下移动的过程中，保证以下情形之一成立：

- a. 如果当前结点的左子结点不是 2- 结点，完成，继续向左下遍历；
- b. 如果当前结点的左子结点是 2- 结点而左子结点的最近的右兄弟结点不是 2- 结点，那么我们从这个兄弟结点处“借”一个结点给左子结点，如下图所示：
    - 我们可以看到，这个借的本质是 左子结点往父结点借，父结点往左子结点的右兄弟结点处借。


- c. 如果当前结点的左子结点和左子结点右边的最近的亲兄弟结点不是 2- 结点，由于这里我们有这个父结点一定不是 2- 结点，因此，我们将这个父结点的最小的 key，左子结点，左子结点的最近的右兄弟结点，合并成一个临时的 4- 结点。

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065926.png)

在遍历的过程中执行这个过程，最后能够得到一个含有最小 key 的 3- 结点或者 4- 结点，然后我们就可以直接从中将其删除，将 3- 结点变为 2- 结点，或者将 4- 结点变为 3- 结点。然后我们再回头向上分解所有临时的 4- 结点。

2-3 树对应到 LLRB，就是要保证在遍历的时候，当前结点是红色，或者当前结点的左子结点是红色。


### 总结

直接实现 2-3 树比较复杂，执行插入、删除操作时要维护和变换的信息比较多，可能性能还不如普通的二叉查找树。

> 尽管我们可以用不同的数据类型表示 2- 结点和 3- 结点并写出变换所需的代码，但用这种直白的表示方法实现大多数的操作并不方便，因为需要处理的情况实在太多。我们需要维护两种不同类型的结点，将被查找的键和结点中的每个键进行比较，将链接和其他信息从一种结点复制到另一种结点，将结点从一种数据类型转换到另一种数据类型，等等。实现这些不仅需要大量的代码，而且它们所产生的额外开销可能会使算法比标准的二叉查找树更慢。

## 左倾红黑树（LLRB）

红黑树的基本思想是用标准的二叉查找树（结点都是 2- 结点），以及一些额外的信息（用来替换 3- 结点）来表示 2-3 树。

我们将树中的 link 分为两类，一类是 red link 将两个 2- 结点连接起来，就构成了一个 **3- 结点**，合法的 LLRB 要求只有父结点和左子结点之间的 link 才能是红色的；另一类是 black link 将两个 2- 结点连接起来，对应 2-3 树中的普通 link。

准确说，我们将 **3- 结点**表示为由一个**左斜的 red link** 相连的两个 2- 结点，如所示，我们可以直接使用标准 BST 的 `get` 方法。

LLRB 的另一种定义是含有 red 和 black link 并且满足下列条件的二叉查找树：

- red link 均为左斜 link；
- 没有一个结点同时和两条 red link 相连；
- 该树是**完美黑色平衡**的，即任意 null 结点到根结点的路径上的 black link 数量相同。

如果我们将一颗 LLRB 中的 red link 画平，那么所有 null 结点到根结点的距离都将是相同的。

由于每个结点都只有一个父结点，因此对每个结点来说，从它的父结点到它自身的 link 是唯一的，因此我们可以将 link 的颜色保存再表示结点的类 Node 的布尔成员变量 color 中。我们规定 null link 是黑色的

```cpp
const bool kRed = true;
const bool kBlack = false;
class Node {
  private:
    int key_;
    int val_;
    int cnt_;
    bool color_;
    Node *left_;
    Node *right_;

  public:
    Node(int key, int val, int cnt, bool color) :
        key_(key), val_(val), cnt_(cnt), color_(color), left_(nullptr), right_(nullptr) {
    }

  private:
    bool isRed(Node *x) {
        if (x == nullptr) {
            return false;
        }
        return x->color_ == kRed;
    }
};
```

### LLRB 的旋转

当我们执行插入结点到 LLRB 中或者删除 LLRB 中的结点时，可能会出现操作后的树不满足 LLRB 的限制条件的情况。这时候，我们就需要通过**旋转**操作使这棵树重新满足 LLRB 的约束条件。

旋转可以分为左旋和右旋。

- 假设我们说左旋结点 E，即结点 E 绕它的右子结点 S **向左**（即**逆时针**）旋转，使得 E 成为 S 的左儿子。由于此时 S 可能会有三个子结点，因此，我们要让 S 原来的左子结点变成新位置的 E 的右子结点；
- 假设我们说右旋结点 S，即结点 S 绕它的**左子结点** E **向右**（即**顺时针**）旋转，使得 S 成为 E 的右儿子，此时 E 可能会有三个子结点，因此，我们要让 E 原来的右子结点变成新位置的 S 的左子结点。

上面说的是普通的二叉树的左旋和右旋，对于红黑树来说，我们在进行旋转时，还要进行**染色**操作。即 E 和 S 的颜色要进行互换：

如下图所示：

![Row2YuW65gySdXf](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065930.png)

C++ 的实现代码如下，以左旋为例，之前我们说了，E 和 S 的颜色要进行互换，但是，S 必然是 RED 才会触发旋转操作，因此我们执行 `rson->color_ = node->color_;` 后，直接将 `node->color_` 置为了 `kRed`。

```cpp
auto RotateLeft(Node *node) {
    auto *rson = node->right_;
    node->right_ = rson->left_;
    rson->left_ = node;
    rson->color_ = node->color_;
    node->color_ = kRed;
    return rson;
}

auto RotateRight(Node *node) {
    auto *lson = node->left_;
    node->left_ = lson->right_;
    lson->right_ = node;
    lson->color_ = node->color_;
    node->color_ = kRed;
    return lson;
}
```

旋转操作可以完美保持 LLRB 的两个重要性质：**有序性**和**完美平衡性**。

### LLRB 的插入操作

我们在执行插入操作时，被插入的结点**总是与父结点使用 red link 来连接**，这样才有可能维护 LLRB 的完美平衡性。（与 2-3 树对应）

#### 1. 向只有一个 2- 结点的树中插入新 key

这里我们考虑的是树中只有一个结点 `h`的情况，然后推广到一般情况：

- 如果 `key < h->key_`，那么这个结点作为左子结点，新的 LLRB 和 单个 3- 结点完全等价，不会破坏 LLRB 的完美平衡；
- 否则，`key > h->key_`，出现了右斜 link 为 red link 的情况，因此我们需要对父结点执行一次左旋，即 `root = RotateLeft(root);`。

#### 2. 向树底部的 2- 结点插入新 key

与情况 1. 其实是一样的，如下图：

![nSNiZXdksPgWjRI](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065933.png)

#### 3. 向一个双 key 树（即 3- 结点）中插入新 key

先考虑只有两个结点的情况：

- a. 新 key 大于原来 3- 结点中的两个 key，那么这个 key 插入作为根结点的右子结点；如果此时我们将两个 link 的颜色都由红染黑，那么就得到了一个由 3 个结点组成、高为 2 的完美平衡的 2-3 树，也就是完美平衡的 LLRB；

![8R3ZAmDcsiEXMx7](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065935.png)

```cpp
void FlipColors(Node *x) {
    x->color_ = kRed;
    x->left_->color_ = kBlack;
    x->right_->color_ = kBlack;
}
```

- b. 新 key 小于原来 3- 结点中的两个 key，那么新 key 插入作为左子结点的左子结点，那么就出现了两个连续的 red link，此时我们只需要将根结点右旋，即可转化为情况 a；

![h2ur3BYiIZ6CsKp](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065936.jpg)

- c. 新 key 介于原来 3- 结点的两个 key 中间，那么新 key 插入作为左子结点的右子结点，此时存在右倾的 red link，我们对根结点的左子结点执行左旋操作，则转化为情形 b，情形 b 可以再转化为情形 a。

![WuYzRXKxVvZfQy1](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065938.jpg)

注意情况 a. 中，结点 b 已经是**根结点**了，所以我们将 b 染成 黑色，否则需要向上传递（即将 b 染成红色），见后续对插入的讨论。

#### 4. 向树底部的 3- 结点插入新键

假设我们需要在树底部的一个 3- 结点下插入一个新结点，那么 a. b. c. 三种情况都可能会出现，在经过一次或者两次旋转之后，都会转换到情形 a.，只是结点 "b" 不再是根结点了，事实上 "b" 本来应该是红色才对，只是情形 a. 中，"b" 是根结点，因此最后染成了黑色。

在 4. 中，结点 "b" 被染成了红色，相当于在 "b" 的父结点中插入了一个红色结点，这就**递归**到了情形 1. 或者 情形 2. 或者情形 3.。即我们相当于在一级一级地往上传递 red link，直到可以满足 LLRB 的要求。

在一级一级地往上传递 red link 时，即在沿着插入点到根结点的路径向上移动时经过的每个结点中顺序完成以下操作，我们就能完成插入操作：

- 如果结点 x 的左子结点是红色的而右子结点是黑色的，对 x 执行**左旋**操作；
- 如果结点 x 的左子结点是红色的，并且左子结点的左子结点也是红色的，对 x 执行 **右旋**操作；
- 如果结点 x 的左子结点和右子结点都是红色的，对 x 执行 `FlipColors` 操作；

### LLRB 的插入操作的实现

通过递归，我们可以以一个比较优雅的方式实现 LLRB 的插入操作，尽量减少了需要分类讨论的情形：

```cpp
const bool kRed = true;
const bool kBlack = false;
class Node {
  private:
    int key_;
    int val_;
    int cnt_;
    bool color_;
    Node *left_;
    Node *right_;
    Node *root_;

  public:
    Node(int key, int val, int cnt, bool color) :
        key_(key), val_(val), cnt_(cnt), color_(color), left_(nullptr), right_(nullptr) {
    }
    void Put(int key, int val) {
        root_ = Put(key, val, root_);
        root_->color_ = kBlack; // 根结点的颜色一定是黑色！
    }

  private:
    static auto isRed(Node *x) -> bool {
        if (x == nullptr) {
            return false;
        }
        return x->color_ == kRed;
    }
    auto RotateLeft(Node *node) {
        auto *rson = node->right_; // 右子结点
        node->right_ = rson->left_;
        rson->left_ = node;
        rson->color_ = node->color_;
        node->color_ = kRed;
        return rson;
    }

    auto RotateRight(Node *node) {
        auto *lson = node->left_; // 左子结点
        node->left_ = lson->right_;
        lson->right_ = node;
        lson->color_ = node->color_;
        node->color_ = kRed;
        return lson;
    }

    void FlipColors(Node *node) {
        node->color_ = kRed;
        node->left_->color_ = kBlack;
        node->right_->color_ = kBlack;
    }
    auto Put(int key, int val, Node *root) -> Node * { // root 表示我们往以 root 为根结点的树中插入结点
        if (root == nullptr) {
            return new Node(key, val, 1, kRed);
        }
        if (key < root->key_) {
            root->left_ = Put(key, val, root->left_);
        } else if (key > root->key_) {
            root->right_ = Put(key, val, root->right_);
        } else {
            root->val_ = val;
        }

        if (!isRed(root->left_) && isRed(root->right_)) {
            // 只有右子结点是红色
            root = RotateLeft(root);
        }
        if (isRed(root->left_) && isRed(root->left_->left_)) {
            // 左子结点和左子结点的左子结点都是红色
            root = RotateRight(root);
        }
        if (isRed(root->left_) && isRed(root->right_)) {
            // 左右子结点都是红色，翻转颜色
            FlipColors(root);
        }
        root->cnt_ = root->left_->cnt_ + root->right_->cnt_ + 1;
        return root;
    }
};
```

### LLRB 的删除操作

LLRB 的删除操作比起插入来说要复杂很多。

类似插入操作，我们也需要一系列局部变化来删除一个结点的同时保持树的完美性，这个过程比插入结点更复杂。

> 我们不仅要在（为了删除一个结点而）构造临时 4- 结点时沿着查找路径向下进行变换，还要在分解遗留的 4- 结点时沿着查找路径向上进行变换（同插入操作）。

#### 删除最小 key

~~让我们从 2-3 树的删除最小 key 的步骤来理解这段代码：~~

~~首先是 `DeleteMin()`，如果根结点的左右子结点存在红色结点（事实上只可能左子结点为红色），那么将直接执行到 `DeleteMin(root_->left_)`；而如果根结点的左右子结点均为黑色，即一共三个 2- 结点，那么，2-3 树中，我们需要将这三个 2- 结点合并成一个临时 4- 结点，对应代码中，则是先在 `DeleteMin()` 中将根结点染为红色，然后 `DeleteMin(root_)` 中，执行 `FlipColorsForDel(root_)`，从而将根结点重新染黑，左子结点和右子结点均染为红色，这与合并三个 2- 结点成一个临时 4- 结点是一致的。~~

~~`MoveRedLeft(node)` 做的事情其实不难理解，如果当前结点为红色（并且此时左子结点和右子结点一定都是黑色，右子结点黑色由 2-3 树的定义保证），如果右子结点是 2- 结点，即右子结点的左子结点也是黑色，那么就合并当前结点、当前结点的左子结点、当前结点的右子结点为一个 4- 结点，对应 `FlipColorsForDel(node)`；否则，当前结点的右子结点是个 3- 结点，那么就执行如下操作：~~

![75YLgEIe4sOG3FZ](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065940.jpg)

~~这里的操作类似于 2-3 结点的操作，但是不能完全对应起来，可以说是构建了一个临时的 5- 结点，继续往下删除了 b 之后，就可以将红色的 c 结点下溢，和 d 结点共同组成一个 3- 结点，然后进行递归的归过程，让 2-3 树重新恢复完美平衡。对应代码，就是一直遍历到当前结点为 c 才会进行 `MoveRedLeft(node)`。~~

![nyxsgZDEGJv2ufb](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065943.jpg)

~~然后在归的过程中，由下往上依次执行 `FixUp(node)`，就能让 LLRB 恢复平衡，完成删除。~~

在删除最小 key 的过程中，我们其实在维护一个隐含的不变量，那就是 `h` 或者 `h->left_` 其中之一一定是红色。因为这样能保证到最终要删除的那个叶子结点一定是红色，因此最终可以直接删除最小 key。

> 叶子结点的子结点是 null，一定是黑色，所以叶子结点本身一定是红色。

由于我们要维护 `h` 或者 `h->left_` 为红色这一性质，因此当我们碰到了 `h->left_` 以及 `h->left_->left_` 为黑色这一情形时，就需要额外操作了，即执行 `MoveRedLeft`。

执行 `MoveRedLeft` 时，此时一定有 `h->left_ == kBlack && h->right_ == kBlack`，我们需要分两种情况讨论

- `h->right_->left_ != kRed`，那么我们直接翻转 `h`、`h->left_`、`h->right_` 三个结点的颜色即可；
![8ZLkf2BQcIHsCuh](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065945.png)
- `h->right_->left_ == kRed`，我们还是要执行翻转 `h`、`h->left_`、`h->right_` 三个结点的颜色，但此时，我们注意到出现了一个新的违反平衡的情形（连续两个 Red link），并且这个对平衡的违反无法在归的过程中通过调用 `FixUp` 解决，因此我们还需要额外进行处理，即通过两次旋转，去除 `h->right_` 为根结点的违反平衡的情况，如下图所示：

![PuqXhid2QDV6Apg](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065947.png)

因此，`MoveRedLeft(Node *h)` 的实现代码如下：

```cpp
auto MoveRedLeft(Node *h) -> Node * {
    FlipColors(h);
    if (isRed(h->right_->left_)) {
        h->right_ = RotateRight(h->right_);
        h = RotateLeft(h);
        FlipColors(h); // 这一行也可以不加，留给归过程中的 FixUp 解决
    }
}
```

由于我们只有碰到 `h->left_` 以及 `h->left_->left_` 这一情形，才出现了违背我们要求的性质的情况，此时才需要调用 `MoveRedLeft`，因此也可以很方便写出 `DelMin` 的代码：

```cpp
void DelMin() {
    root_ = DelMin(root_);
    root_->color_ = kBlack;
}

auto DelMin(Node *h) -> Node * {
    if (h->left_ == nullptr) {
        return nullptr; // 删除 h，h 是最小 key
    }
    if (!isRed(h->left_) && !isRed(h->left_->left_)) {
        h = MoveRedLeft(h);
    }
    h = DelMin(h->left_);
    return FixUp(h);
}

auto MoveRedLeft(Node *h) -> Node * {
    FlipColors(h);
    if (isRed(h->right_->left_)) {
        h->right_ = RotateRight(h->right_);
        h = RotateLeft(h);
        FlipColors(h); // 这一行也可以不加，留给归过程中的 FixUp 解决
    }
    return h;
}

auto FixUp(Node *node) -> Node * {
    if (!isRed(node->left_) && isRed(node->right_)) {
        // 只有右子结点是红色
        node = RotateLeft(node);
    }
    if (isRed(node->left_) && isRed(node->left_->left_)) {
        node = RotateRight(node);
    }
    if (isRed(node->left_) && isRed(node->right_)) {
        FlipColors(node);
    }
    node->cnt_ = node->left_->cnt_ = node->right_->cnt_ + 1;
    return node;
}
```

#### 删除最大 key

删除最大 key 的步骤和删除最小 key 类似，我们也需要维护一个性质，即 **`h` 或者 `h->right_` 为红色**。我们可以先观察 `h->left_` 是否是红色，如果是，对 `h` 执行右旋，那么 `h->right_` 一定会变成红色，我们继续往右下遍历即可。当我们往右下遍历时，如果碰到了 `!isRed(h->right_) && !isRed(h->right_->left_)` 的情况时（`h->left_` 不为红色），就需要执行 `MoveRedRight` 操作了，不然就会出现违反性质的情况。

- 如果 `h->left_->left_` 不是红色，那么直接对 `h` 执行 `FlipColors` 即可，翻转了 `h`、`h->left_`、`h->right_` 这三个结点的颜色，如下图：
![Dw34mrGYxClqVIb](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065949.png)

- 如果 `h->left_->left_` 是红色，就像 `DelMin()` 时那样，我们需要翻转 `h`、`h->left_`、`h->right_` 之后，会留下一个后续 `FixUp` 无法处理的不平衡性，即 `h` 左侧有两个连续 red link，同时 `h` 右侧也是 red link，而如果我们对 `h` 执行右旋，那么 `h` 右端就有连续两个 red link，`h->right_` 必定能满足性质。

```cpp
auto MoveRedRight(Node *h) -> Node * {
    FlipColors(h);
    if (isRed(h->left_->left_)) {
        h = RotateRight(h);
        FlipColors(h); // 可以留给后续 FixUp 执行
    }
    return h;
}
```

在 `DelMax` 的过程中，一旦碰到 `h->left_` 是红色的情况，我们就执行右旋，一旦碰到 `!isRed(h->right_) && !isRed(h->right_->left_)` 的情况，就执行 `MoveRedRgiht()`。

![2mfOP9HKaZxkniy](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065951.png)

> 之所以是判断 `h->right_->left_`，是因为假如 `h->right_` 是黑色，`h->right_->left_` 是红色，当我们遍历到 `h->right_` 时，可以左旋 `h->right_`，一样可以满足条件；

```cpp
void DelMax() {
    root_ = DelMax(root_);
    if (root_ != nullptr) {
        root_->color_ = kBlack;
    }
}
auto MoveRedRight(Node *h) -> Node * {
    FlipColors(h);
    if (isRed(h->left_->left_)) {
        h = RotateRight(h);
        FlipColors(h); // 可以不加，等后续 FixUp 处理
    }
    return h;
}

auto DelMax(Node *h) -> Node * {
    if (isRed(h->left_)) {
        h = RotateRight(h);
    }
    if (h->right_ == nullptr) {
        return nullptr;
    }
    if (!isRed(h->right_) && !isRed(h->right_->left_)) { // 
        h = MoveRedRight(h);
    }
    h->right_ = DelMax(h->right_);
    return FixUp(h);
}
```


#### 删除任意结点

我们首先考虑删除叶子结点，与删除最小值类似，我们在删除任意值的过程中，也要维护一个性质，**`h` 或者 `h` 的左子结点或者右子结点是红色的**。

准确的说，当我们经过比较之后，发现要向左，那么就要求 `h->left_` 和 `h->left_->left_` 不能都是黑色（`h->left_->right_` 一定是黑色），不然当我们移动到 `h->left_` 时，就会发现性质已经不满足了；

```cpp
auto Delete(int key, Node *h) -> Node * {
    if (key < h->key_) {
        if (!isRed(h->left_) && !isRed(h->left_->left_)) {
            h = MoveRedLeft(h);
        }
        h->left_ = Delete(key, h->left_);
    }
    return FixUp(h);
}
```

当我们比较之后，发现可能要向右时，这时候情况比较复杂，就像 `DelMax` 中那样，如果 `h->left_` 时红色，那我们先对 `h` 执行右旋，这样如果我们碰到 `key == h->key_ && h->right_ == nullptr` 时，就能直接删除 `h`（事实上如果 `h->left_` 不为红色，不可能出现 `h` 是黑色且 `h->right_` 是黑色的情况，此时 `h` 一定是红色的叶子结点了）；

之后，我们先判断 `!isRed(h->right_) && !isRed(h->right_->left_)`，如果为 `true`，执行 `MoveRedRight(h)`，从而保证往右时，不会出现 `h`、`h->left_`、`h->right_` 均为黑色的情况。

然后如果，`key == h->key_`，那么我们将当前结点的的后继结点的 key-value 赋值给当前结点，然后删除右子树的最小结点；

> 结点 `h` 的后继结点是大于 `h->key_` 的最小 key 对应的结点，亦即 `h` 的右子树的最小结点。

最后剩下的情况，我们继续向右处理删除操作。

```cpp
auto Delete(int key, Node *h) -> Node * {
    if (key < h->key_) {
        if (!isRed(h->left_) && !isRed(h->left_->left_)) {
            h = MoveRedLeft(h);
        }
        h->left_ = Delete(key, h->left_);
    } else {
        if (isRed(h->left_)) {
            h = RotateRight(h);
        }
        if (key == h->key_ && h->right == nullptr) {
            return nullptr;
        }
        if (!isRed(h->left_) && !isRed(h->right_->left_)) {
            h = MoveRedRight(h);
        }
        // 经过这些操作，h->key_ 只会变小，不会变大，因此不可能出现 key < h->key_ 的情况
        if (key == h->key_) {
            h->key_ = GetMin(root->right_)->key_;
            h->value_ = GetMin(root->right_)->value_;
            h->right_ = DelMin(root->right_);
        } else {
            h->right_ = Delete(key, h->right_);
        }
    }
    return FixUp(h);
}
```

### 完整实现

LLRB 的完整实现代码如下：

```cpp
const bool kRed = true;
const bool kBlack = false;
class Node {
  private:
    int key_;
    int val_;
    int cnt_;
    bool color_;
    Node *left_;
    Node *right_;
    Node *root_;

  public:
    Node(int key, int val, int cnt, bool color) :
        key_(key), val_(val), cnt_(cnt), color_(color), left_(nullptr), right_(nullptr) {
    }
    void Put(int key, int val) {
        root_ = Put(key, val, root_);
        root_->color_ = kBlack; // 根结点的颜色一定是黑色！
    }
    void DelMin() {
        root_ = DelMin(root_);
        if (root_ != nullptr) {
            root_->color_ = kBlack;
        }
    }

    void DelMax() {
        root_ = DelMax(root_);
        if (root_ != nullptr) {
            root_->color_ = kBlack;
        }
    }

    auto Delete(int key, Node *h) -> Node * {
        if (key < h->key_) {
            if (!isRed(h->left_) && !isRed(h->left_->left_)) {
                h = MoveRedLeft(h);
            }
            h->left_ = Delete(key, h->left_);
        } else {
            if (isRed(h->left_)) {
                h = RotateRight(h);
            }
            if (key == h->key_ && h->right_ == nullptr) {
                return nullptr;
            }
            if (!isRed(h->left_) && !isRed(h->right_->left_)) {
                h = MoveRedRight(h);
            }
            // 经过这些操作，h->key_ 只会变小，不会变大，因此不可能出现 key < h->key_ 的情况
            if (key == h->key_) {
                h->key_ = GetMin(h->right_)->key_;
                h->val_ = GetMin(h->right_)->val_;
                h->right_ = DelMin(h->right_);
            } else {
                h->right_ = Delete(key, h->right_);
            }
        }
        return FixUp(h);
    }

  private:
    auto GetMin(Node *x) -> Node * {
        if (x->left_ == nullptr) {
            return x;
        }
        return GetMin(x->left_);
    }
    static auto isRed(Node *x) -> bool {
        if (x == nullptr) {
            return false;
        }
        return x->color_ == kRed;
    }
    static auto RotateLeft(Node *node) -> Node * {
        auto *rson = node->right_;
        node->right_ = rson->left_;
        rson->left_ = node;
        rson->color_ = node->color_;
        node->color_ = kRed;
        return rson;
    }

    static auto RotateRight(Node *node) -> Node * {
        auto *lson = node->left_;
        node->left_ = lson->right_;
        lson->right_ = node;
        lson->color_ = node->color_;
        node->color_ = kRed;
        return lson;
    }

    static void FlipColors(Node *node) {
        node->color_ = !node->color_;
        node->left_->color_ = !node->left_->color_;
        node->right_->color_ = !node->right_->color_;
    }
    auto Put(int key, int val, Node *node) -> Node * { // node 表示我们往以 node 为根结点的树中插入结点
        if (node == nullptr) {
            return new Node(key, val, 1, kRed);
        }
        if (key < node->key_) {
            node->left_ = Put(key, val, node->left_);
        } else if (key > node->key_) {
            node->right_ = Put(key, val, node->right_);
        } else {
            node->val_ = val;
        }

        if (!isRed(node->left_) && isRed(node->right_)) {
            // 只有右子结点是红色
            node = RotateLeft(node);
        }
        if (isRed(node->left_) && isRed(node->left_->left_)) {
            node = RotateRight(node);
        }
        if (isRed(node->left_) && isRed(node->right_)) {
            FlipColors(node);
        }
        node->cnt_ = node->left_->cnt_ + node->right_->cnt_ + 1;
        return node;
    }

    auto FixUp(Node *node) -> Node * {
        if (!isRed(node->left_) && isRed(node->right_)) {
            // 只有右子结点是红色
            node = RotateLeft(node);
        }
        if (isRed(node->left_) && isRed(node->left_->left_)) {
            node = RotateRight(node);
        }
        if (isRed(node->left_) && isRed(node->right_)) {
            FlipColors(node);
        }
        node->cnt_ = node->left_->cnt_ = node->right_->cnt_ + 1;
        return node;
    }
    auto MoveRedLeft(Node *h) -> Node * {
        // 假定 node 是红色，我们的删除结点的代码也能够保证这一点；
        // 同时 node->left_ 和 node->left_->left_ 都是黑色
        FlipColors(h);
        if (isRed(h->right_->left_)) {
            h->right_ = RotateRight(h->right_);
            h = RotateLeft(h);
            FlipColors(h); // 可以不加，等后续处理
        }
        return h;
    }
    auto MoveRedRight(Node *h) -> Node * {
        FlipColors(h);
        if (isRed(h->left_->left_)) {
            h = RotateRight(h);
            FlipColors(h); // 可以不加，等后续 FixUp 处理
        }
        return h;
    }

    auto DelMin(Node *node) -> Node * {
        if (node->left_ == nullptr) {
            return nullptr;
        }
        if (!isRed(node->left_) && !isRed(node->left_->left_)) {
            node = MoveRedLeft(node);
        }
        node->left_ = DelMin(node->left_);
        return FixUp(node);
    }
    auto DelMax(Node *h) -> Node * {
        if (isRed(h->left_)) {
            h = RotateRight(h);
        }
        if (h->right_ == nullptr) {
            return nullptr;
        }
        if (!isRed(h->right_) && !isRed(h->right_->left_)) {
            // 之所以是判断 h->right_->left_，是因为假如 h->right_ 是黑色，h->right_->left_ 是红色
            // 当我们遍历到 h->right_ 时，可以左旋 h->right_，一样可以满足条件；
            h = MoveRedRight(h);
        }
        h->right_ = DelMax(h->right_);
        return FixUp(h);
    }
};
```