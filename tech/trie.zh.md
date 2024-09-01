---
title: "字典树"
date: 2023-05-29T09:54:58+08:00
lastmod: 2023-05-29T09:54:58+08:00 #更新时间
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
## 定义
字典树（Trie），是一个像字典一样的树，又称前缀树。

可以高效的查询某个字符串是否在一组给定的字符串中，或者说查询某个单词是否在字典中。

字典树的查询时间复杂度可以认为是 $O(l)$，其中 $l$ 为待查询单词的长度。

## 引入
字典树示意图：

![bAGz3DrkXeP5vch](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065821.jpg)

可以发现，这棵字典树用边来代表字母，而根结点到树上面某一个节点的路径就代表一个字符串，例如 $1\rightarrow 4\rightarrow 8\rightarrow 12$ 表示的就是字符串 `caa`，如果结点 $12$ 对应的 `end_` 字段的值为 $true$，说明 `caa` 是字典树中一个完整的字符串，否则只是一个前缀。

trie 的结构非常好懂，我们用 $\delta(u,c)$ 表示结点 $u$ 的 $c$ 字符指向的下一个结点，或着说是结点 $u$ 代表的字符串后面添加一个字符 $c$ 形成的字符串的结点。（ $c$ 的取值范围和字符集大小有关，不一定是 $0\sim 26$ 。）

## 字典树的实现
这里放一个简单的前缀树的类的实现，
```cpp
struct Trie {
  int nex[10000][26], cnt; // nex 的第一维度表示前缀树的结点数量，与上面的图相对应
  bool end[10000];  // 该结点结尾的字符串是否存在

  void insert(char *s, int l) {  // 插入字符串
    int p = 0;
    for (int i = 0; i < l; i++) {
      int c = s[i] - 'a';
      if (!nex[p][c]) nex[p][c] = ++cnt;  // 如果没有，就添加结点
      p = nex[p][c];
    }
    exist[p] = 1;
  }

  bool find(char *s, int l) {  // 查找字符串
    int p = 0;
    for (int i = 0; i < l; i++) {
      int c = s[i] - 'a';
      if (!nex[p][c]) return 0;
      p = nex[p][c];
    }
    return exist[p];
  }
};
```

也可以使用 `new` 动态创建前缀树，例如：
```cpp
class Trie {
  private:
    vector<Trie *> vec;
    bool is_end;

  public:
  	Trie() : vec(26), is_end(false) {
        for (int i = 0; i < 26; ++i) {
  			vec[i] = nullptr;
  		}
  	}
  	void insert(string word) {
  		Trie *node = this;
  		for (int i = 0; i < word.size(); ++i) {
  			if (node->vec[word[i] - 'a'] != nullptr) {
  				node = node->vec[word[i] - 'a'];
  			} else {
  				node->vec[word[i] - 'a'] = new Trie();
  				node = node->vec[word[i] - 'a'];
  			}
  		}
  		node->is_end = true;
  	}
}
```

## 应用
### 字典查找
字典树的最经典的应用——查找一个字符串是否在“字典”中出现过。

[745. Prefix and Suffix Search (Hard)](https://leetcode.com/problems/prefix-and-suffix-search/)

[676. Implement Magic Dictionary (Medium)](https://leetcode.com/problems/implement-magic-dictionary/)

[212. Word Search II (Hard)](https://leetcode.com/problems/word-search-ii/)

[648. Replace Words (Medium)](https://leetcode.com/problems/replace-words/)

### 01 字典树
将数的二进制表示看成一个字符串，就可以构建出字符集为 $\{0, 1\}$ 的字典树。

可以用于维护异或极值或者异或和。
[421. Maximum XOR of Two Numbers in an Array (Medium)](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/)



## 参考
[OI-Wiki](https://oi-wiki.org/string/trie/)

