---
title: "无向图形式组织的树"
date: 2023-07-18T09:30:57+08:00
lastmod: 2023-07-18T09:30:57+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["tree", "graph", "data structure and algorithms"]
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
如 [数组形式组织的树](https://blog.zwyyy456.tech/zh/posts/tech/tree_in_array/) 中所说，树一般以链表结点的形式组织，定义如下：

```cpp
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x): val(x), left(nullptr), right(nullptr) {}
};
```

也可能以数组的形式组织，即使用 $parent$ 数组，$y = parent[x]$ 说明 $y$ 是 $x$ 的父结点，根结点的父结点为 $-1$，表示父结点不存在。

还可以使用无向图的形式来表示，例如 leetcode 的 [834. 树中距离之和](https://leetcode.cn/problems/sum-of-distances-in-tree/)。

昨天做这个题的时候，整体思路挺好想的，但就是有个地方被困住了，那就是，在树的无向图的表示情况下，如何统计以当前结点为根结点的子树的数量？（没办法转化成有向图！）

## 统计以当前结点为根结点的子树的结点数

统计方法还是深度优先搜索（dfs），只不过，相比一般的深度优先搜索，我们需要传入一个额外的参数，即上一次搜索的父结点，如下图所示：

![DwMF7gBc6HXJybs](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065832.jpg)

相应的 dfs 代码为

```cpp
for (int child : tree[pa]) {
    if (child == ancestor) {
        continue;
    }
    // 对子结点进行 dfs ...
}
```

这样就**确定出了一个遍历方向**，因此，整体思路就是，我们可以任意选择一个结点作为 dfs 的起点（这里就选择 $0$ 号结点），依次进行 dfs，利用递归的方法，统计以当前结点为根结点的子树的结点数。

因此，[834. 树中距离之和](https://leetcode.cn/problems/sum-of-distances-in-tree/) 的完整解题代码如下：

```cpp
class Solution {
  public:
    int count(vector<vector<int>> &tree, vector<int> &dis, vector<int> &cnt, int pa, int grandpa) {
        int res = 1;
        for (int child : tree[pa]) {
            if (child == grandpa) { // 防止重复遍历，保证 dfs 遍历时的单向性
                continue;
            }
            dis[child] = dis[pa] + 1;
            res += count(tree, dis, cnt, child, pa);
        }
        cnt[pa] = res;
        return res;
    }
    vector<int> sumOfDistancesInTree(int n, vector<vector<int>> &edges) {
        vector<vector<int>> tree(n);
        for (auto &vec : edges) {
            tree[vec[0]].push_back(vec[1]);
            tree[vec[1]].push_back(vec[0]); // 注意，建立无向图要 push_back 两次！
        }
        vector<int> cnt(n);
        vector<int> dp(n);
        vector<int> dis(n); // 表示结点 0 到其他结点的最短距离
        count(tree, dis, cnt, 0, -1);
        for (int i = 0; i < n; ++i) {
            dp[0] += dis[i];
        }
        queue<pair<int, int>> q;
        q.push({0, -1}); // pa, grandpa
        while (!q.empty()) {
            auto [pa, grandpa] = q.front();
            q.pop();
            for (int child : tree[pa]) {
                if (child == grandpa) { // 保证 bfs 遍历时的单向性
                    continue;
                }
                dp[child] = dp[pa] + n - 2 * cnt[child];
                q.push({child, pa});
            }
        }
        return dp;
    }
};
```


