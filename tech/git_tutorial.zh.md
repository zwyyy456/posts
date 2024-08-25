---
title: "git 使用技巧"
date: 2023-03-22T09:56:48+08:00
lastmod: 2023-03-22T09:56:48+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["git", "geek", "utility", "tips"]
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
## 设置默认编辑器为 vim
```
git config --global core.editor vim
```

## 问题`fatal: in unpopulated submodule 'xxx'`的解决
出现这个问题的原因clone的别人的项目之后，删除项目里的`.git`文件就直接添加到了自己的版本控制里面，解决方案，执行
`git rm --cached . -rf`，再添加文件和提交。

## "merge conflict" 解决
对于纯文本文件的冲突解决，可以参考[该文章](https://www.liaoxuefeng.com/wiki/896043488029600/900004111093344)，没什么太多好说的。

对于二进制文件，编辑二进制文件来解决冲突是不现实的，要么选择对方的修改，要么选择自己的修改，可以使用`git checkout`的`--theirs`或者`--ours`选项：
```sh
git pull
git checkout --theirs YOUR_BINARY_FILE
// git checkout --ours YOUR_BINARY_FILE
git add YOUR_BINARY_FILE
git commit -m 'merged with the remote repos.'
git push
```

