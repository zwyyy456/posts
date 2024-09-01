---
title: "数组形式组织的树"
date: 2023-06-12T13:47:07+08:00
lastmod: 2023-06-12T13:47:07+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["tree", "data structure and algorithms"]
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
## 引入

在 LeetCode 中，二叉树一般是以链表结点的形式组织的，定义如下：

```cpp
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x): val(x), left(nullptr), right(nullptr) {}
};
```

其实也可以用数组的形式组织，即使用 $parent$ 数组，$y = parent[x]$ 说明 $y$ 是 $x$ 的父结点，根结点的父结点为 $-1$，表示父结点不存在。

## 最近公共祖先

### 链表形式

对链表形式树，求最近公共祖先可以使用递归很方便的解决：

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (root == nullptr || root == p || root == q) {
            return root;
        }
        TreeNode *left = lowestCommonAncestor(root->left, p, q);
        TreeNode *right = lowestCommonAncestor(root->right, p, q);
        if (left != nullptr && right != nullptr) {
            return root;
        }
        if (left != nullptr && right == nullptr) {
            return left;
        }
        if (left == nullptr && right != nullptr) {
            return right;
        }
        return nullptr;
    }
};
```

也可以这样写，更好理解，与上面的写法的含义是一样的：

```cpp
class Solution {
  public:
    pair<bool, bool> Ancestor(TreeNode *root, TreeNode *p, TreeNode *q, TreeNode **res) {
        if (root == nullptr) {
            return {false, false};
        }
        auto flagp = root == p, flagq = root == q;
        auto [pleft, qleft] = Ancestor(root->left, p, q, res);
        auto [pright, qright] = Ancestor(root->right, p, q, res);
        bool res_p = flagp || pleft || pright;
        bool res_q = flagq || qleft || qright;
        // 最近公共祖先的条件
        // 左子树，右子树都不是最近公共祖先
        if (res_p && res_q && !(pleft && qleft) && !(pright && qright)) {
            *res = root;
        }
        return {pleft || pright || flagp, qleft || qright || flagq};
    }
    TreeNode *lowestCommonAncestor(TreeNode *root, TreeNode *p, TreeNode *q) {
        TreeNode *res = nullptr;
        Ancestor(root, p, q, &res);
        return res;
    }
};
```

### 数组形式
对数组形式的树来说，求二叉树的最近公共祖先则要复杂很多。

我们需要以 [1483. 树节点的第 K 个祖先 (Hard)](https://leetcode.cn/problems/kth-ancestor-of-a-tree-node/) 的解法作为先决条件，即先处理 $dp[i][j]$ 数组，其中 $dp[i][j]$ 表示编号 $i$ 结点的第 $2^j$ 个祖先的编号，则有递推公式：
$dp[i][j] = dp[dp[i][j - 1]][j - 1]$。

处理好 $dp[i][j]$ 数组之后，我们可以用类似二分查找的办法确定编号为 $m$ 和编号为 $n$ 的结点的深度，这里深度可以定义为满足 $pars[i][k] = -1$ 的最小的 $k$，其中 $pars[i][k]$ 表示编号为 $i$ 结点的第 $k$ 个祖先的编号，可以由 $dp[i][j]$ 在 $O(\log n)$ 时间内求得。

对于求解公共祖先，其实我们也是用二分查找的思想求解，假设 $m$ 结点的深度为 $d_m$，$n$ 结点的深度为 $d_n$，且 $d_m >= d_n$（如果不满足，交换一下两个结点即可），那么等同于求 $n$ 的第 $k$ 个祖先 $p_n$ 以及 $m$ 的第 $k + d_m - d_n$ 个祖先 $p_m$ ，它们满足 $p_m = p_n$ 并且对应的 $k$ 最小。

预处理 $dp[i][j]$ 数组的时间复杂度为 $O(n\log n)$，总的时间复杂度为 $O(n\log n)$。



