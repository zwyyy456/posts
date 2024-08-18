---
title: "计算机教育缺失的一课：Git"
date: 2024-06-16
lastmod: 2024-06-18 #更新时间
author: ["zwyyy456"] #作
categories: ["notes"]
tags: ["mit", "git"]
description: "" #描述
weight: # 输入 1 可以顶置文章，默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---

## 版本控制系统介绍

版本控制系统 (VCSs) 是一类用于追踪源代码（或其他文件、文件夹）改动的工具。顾名思义，这些工具可以帮助我们管理代码的修改历史；不仅如此，它还可以让协作编码变得更方便。VCS 通过一系列的快照将某个文件夹及其内容保存了起来，每个快照都包含了文件或文件夹的完整状态。同时它还维护了快照创建者的信息以及每个快照的相关信息等等。

版本控制系统的事实标准是 Git。

Git 的许多操作或者说命令看起来非常奇怪，但 Git 的底层设计与思想非常优雅，因此，从 Git 的数据模型开始学习 Git，自底向上，最后再学习 Git 的接口或者说命令，会比较容易让人理解 Git 的命令以及 Git 是如何操作数据模型的。

## Git 的数据模型

Git 将项目的根目录（顶层目录）中的文件夹与文件作为集合，通过这一系列集合的快照来管理项目历史记录。在 Git 的术语中，文件被称为 Blob（数据对象），即一组数据，目录则是被称为 `tree`。tree 的名称与另一个 tree 又或者文件相对应。

> 把目录视为 tree，那么子目录就是 subtree，目录下的文件就是 tree 的子节点。

一棵 tree 看起来可能是这样的：

```txt
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

## Git 历史记录建模：关联快照

Git 中的 object 可以分为 blob、tree、commit 三类，每次我们执行 `git commit` 时，都会创建一个 commit 对象，又或者说对当前的 `work directory` 的 snapshot。

> 此后，在本文中不再区分作为名词的 commit 与 snapshot。

在 Git 中，历史记录是一个由 snapshot（commit）组成的有向无环图。Git 中每次 commit 都有一系列的 parent，即当前 commit 之前的一系列 commit，commit 可能有多个 parent，例如当两条分支合并为一条分支时。

以可视化的方式查看这些 commit histories 时，看起来差不多是这样的：

```txt
o <-- o <-- o <-- o
            ^  
             \
              --- o <-- o
```

其中 `o` 表示一次 commit。箭头指向了当前 commit 的 parent，在第三次 commit 之后，历史记录分岔成了两条独立的分支。这可能因为此时需要同时开发两个不同的特性，它们之间是相互独立的。开发完成后，这些分支可能会被合并并创建一个新的 commit，这个新的 commit 会同时包含这些特性。新的 commit 会创建一个新的历史记录，看上去像这样（最新的合并提交用大写 O 标记）：

```txt
o <-- o <-- o <-- o <---- O
            ^            /
             \          v
              --- o <-- o
```

Git 中 commit 发生后，该 commit 是不可变的，修改内容会导致新的 commit 产生，而之前的 commit 仍然不变。

## 数据模型与伪代码表示

以下这段“伪代码”可以更好的理解 Git 的数据模型：

```txt

// 文件就是一组数据
type blob = array<byte>

// 目录可以包含目录，也可以同时包含文件
type tree = map<string, tree | blob>

// 每个 commit 包含该 commit 的 parents，一些元数据，以及顶层目录对应的 tree
type commit = struct {
    parrents: array<commit>
    author: string
    message: string
    snapshot: tree
}
```

## 对象与内存寻址

Git 中的对象可以是 blob、tree 或 commit：

```
type object = blob | tree | commit
```

Git 在储存数据时，所有的对象都会基于它们的 [SHA-1 哈希](https://en.wikipedia.org/wiki/SHA-1) 进行寻址。

```
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```

Blobs、tree 和 commit 都一样，它们都是对象。当它们引用其他对象时，它们并没有真正的在硬盘上保存这些对象，而是仅仅保存了它们的哈希值作为引用。

可以执行 `git cat-file -p <sha-1>` 来查看底层数据的表示：

![6rPqwN](https://pic-upyun.zwyyy456.tech/uPic/6rPqwN.png)

我们首先查看了 commit 的底层数据，可以看到 commit 中存储了 author、commiter、commit-message 等元数据，还存储了顶层目录对应的 tree 的哈希值。

而通过 `git cat-file -p aec8d2` 则可以查看该 tree 的内容，包含了两个 blob 的哈希值，一个是 `animal.py` 的，一个是  `hello.txt` 的。以同样的方案可以查看 blob 的内容：

![V7UaKX](https://pic-upyun.zwyyy456.tech/uPic/V7UaKX.png)

## 引用

所有的 commit 都可以通过其 SHA-1 哈希值来标记，但使用哈希值非常不方便，针对这一问题，Git 的解决方法是给这些哈希值赋予人类可读的名字，也就是引用（references）。引用是指向提交的指针。与对象不同的是，它是可变的（引用可以被更新，指向新的 commit），例如 `master` 引用通常指向主分支的最新一次 commit。

```txt
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
```

有一个细节需要我们注意，通常情况下，我们会想要知道“我们当前所在位置”，并将其标记下来。这样当我们创建新的 snapshot 的时候，我们就可以知道它的相对位置（如何设置它的“parent”）。在 Git 中，我们当前的位置有一个特殊的索引，它就是 "HEAD"。

## 仓库 (repository)

最后，我们可以粗略地给出 Git 仓库的定义了：`对象` 和 `引用`。

> 也许可以再加上对象的哈希值

在硬盘上，Git 仅存储对象和引用：因为其数据模型仅包含这些东西。所有的 `git` 命令都对应着对 commit-tree 的操作，例如增加对象，增加或删除引用。


## 暂存区

Git 中还包括一个和数据模型完全不相关的概念，但它确是创建提交的接口的一部分。

就上面介绍的 snapshot 系统来说，您也许会期望它的实现里包括一个“创建 snapshot”的命令，该命令能够基于当前工作目录的当前状态创建一个全新的 snapshot。有些版本控制系统确实是这样工作的，但 Git 不是。我们希望简洁的 snapshot，而且每次从当前状态创建 snapshot 可能效果并不理想。例如，考虑如下场景，您开发了两个独立的特性，然后您希望创建两个独立的 commit，其中第一个 commit 仅包含第一个特性，而第二个提交仅包含第二个 commit。或者，假设您在调试代码时添加了很多打印语句，然后您仅仅希望提交和修复 bug 相关的代码而丢弃所有的打印语句。

Git 处理这些场景的方法是使用一种叫做“暂存区（staging area）”的机制，它允许您指定下次 snapshot 中要包括那些改动。

`git add xxx` 就表示将 `xxx` 文件添加到暂存区，下次执行 `git commit` 时就会为暂存区的内容创建 snapshot。

## Git 的命令行接口

### 基础


<!-- 
The `git init` command initializes a new Git repository, with repository
metadata being stored in the `.git` directory:

```console
$ mkdir myproject
$ cd myproject
$ git init
Initialized empty Git repository in /home/missing-semester/myproject/.git/
$ git status
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

How do we interpret this output? "No commits yet" basically means our version
history is empty. Let's fix that.

```console
$ echo "hello, git" > hello.txt
$ git add hello.txt
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   hello.txt

$ git commit -m 'Initial commit'
[master (root-commit) 4515d17] Initial commit
 1 file changed, 1 insertion(+)
 create mode 100644 hello.txt
```

With this, we've `git add`ed a file to the staging area, and then `git
commit`ed that change, adding a simple commit message "Initial commit". If we
didn't specify a `-m` option, Git would open our text editor to allow us type a
commit message.

Now that we have a non-empty version history, we can visualize the history.
Visualizing the history as a DAG can be especially helpful in understanding the
current status of the repo and connecting it with your understanding of the Git
data model.

The `git log` command visualizes history. By default, it shows a flattened
version, which hides the graph structure. If you use a command like `git log
--all --graph --decorate`, it will show you the full version history of the
repository, visualized in graph form.

```console
$ git log --all --graph --decorate
* commit 4515d17a167bdef0a91ee7d50d75b12c9c2652aa (HEAD -> master)
  Author: Missing Semester <missing-semester@mit.edu>
  Date:   Tue Jan 21 22:18:36 2020 -0500

      Initial commit
```

This doesn't look all that graph-like, because it only contains a single node.
Let's make some more changes, author a new commit, and visualize the history
once more.

```console
$ echo "another line" >> hello.txt
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   hello.txt

no changes added to commit (use "git add" and/or "git commit -a")
$ git add hello.txt
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        modified:   hello.txt

$ git commit -m 'Add a line'
[master 35f60a8] Add a line
 1 file changed, 1 insertion(+)
```

Now, if we visualize the history again, we'll see some of the graph structure:

```
* commit 35f60a825be0106036dd2fbc7657598eb7b04c67 (HEAD -> master)
| Author: Missing Semester <missing-semester@mit.edu>
| Date:   Tue Jan 21 22:26:20 2020 -0500
|
|     Add a line
|
* commit 4515d17a167bdef0a91ee7d50d75b12c9c2652aa
  Author: Anish Athalye <me@anishathalye.com>
  Date:   Tue Jan 21 22:18:36 2020 -0500

      Initial commit
```

Also, note that it shows the current HEAD, along with the current branch
(master).

We can look at old versions using the `git checkout` command.

```console
$ git checkout 4515d17  # previous commit hash; yours will be different
Note: checking out '4515d17'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 4515d17 Initial commit
$ cat hello.txt
hello, git
$ git checkout master
Previous HEAD position was 4515d17 Initial commit
Switched to branch 'master'
$ cat hello.txt
hello, git
another line
```

Git can show you how files have evolved (differences, or diffs) using the `git
diff` command:

```console
$ git diff 4515d17 hello.txt
diff --git c/hello.txt w/hello.txt
index 94bab17..f0013b2 100644
--- c/hello.txt
+++ w/hello.txt
@@ -1 +1,2 @@
 hello, git
 +another line
```
-->
`git log --all --graph --decorate --oneline` 可以以简洁的有向无环图的方式可视化 Git 历史记录。

- `git help <command>`: 获取 git 命令的帮助信息
- `git init`: 创建一个新的 git 仓库，其数据会存放在一个名为 `.git` 的目录下
- `git status`: 显示当前的仓库状态
- `git add <filename>`: 添加文件到暂存区
- `git commit`: 创建一个新的提交
    - 如何编写 [良好的提交信息](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)!
    - 为何要 [编写良好的提交信息](https://chris.beams.io/posts/git-commit/)
- `git log`: 显示历史日志
- `git log --all --graph --decorate`: 可视化历史记录（有向无环图）
- `git diff <filename>`: 显示与暂存区文件的差异
- `git diff <revision> <filename>`: 显示某个文件两个版本之间的差异
- `git checkout <revision>`: 更新 HEAD 和目前的分支

### 分支和合并

<!--

Branching allows you to "fork" version history. It can be helpful for working
on independent features or bug fixes in parallel. The `git branch` command can
be used to create new branches; `git checkout -b <branch name>` creates and
branch and checks it out.

Merging is the opposite of branching: it allows you to combine forked version
histories, e.g. merging a feature branch back into master. The `git merge`
command is used for merging.

--->

- `git branch`: 显示分支
- `git branch <name>`: 创建分支
- `git checkout -b <name>`: 创建分支并切换到该分支
    - 相当于 `git branch <name>; git checkout <name>`
- `git merge <revision>`: 合并到当前分支
- `git mergetool`: 使用工具来处理合并冲突
- `git rebase`: 将一系列补丁变基（rebase）为新的基线

### 远端操作

- `git remote`: 列出远端
- `git remote add <name> <url>`: 添加一个远端
- `git push <remote> <local branch>:<remote branch>`: 将对象传送至远端并更新远端引用
- `git branch --set-upstream-to=<remote>/<remote branch>`: 创建本地和远端分支的关联关系
- `git fetch`: 从远端获取对象/索引
- `git pull`: 相当于 `git fetch; git merge`
- `git clone`: 从远端下载仓库

### 撤销

- `git commit --amend`: 编辑提交的内容或信息
- `git reset HEAD <file>`: 恢复暂存的文件
- `git checkout -- <file>`: 丢弃修改
- `git restore`: git2.32 版本后取代 git reset 进行许多撤销操作


## Git 高级操作

- `git config`: Git 是一个 [高度可定制的](https://git-scm.com/docs/git-config) 工具
- `git clone --depth=1`: 浅克隆（shallow clone），不包括完整的版本历史信息
- `git add -p`: 交互式暂存
- `git rebase -i`: 交互式变基
- `git blame`: 查看最后修改某行的人
- `git stash`: 暂时移除工作目录下的修改内容
- `git bisect`: 通过二分查找搜索历史记录
- `.gitignore`: [指定](https://git-scm.com/docs/gitignore) 故意不追踪的文件

## 杂项

- **图形用户界面**: Git 的 [图形用户界面客户端](https://git-scm.com/downloads/guis) 有很多，但是我们自己并不使用这些图形用户界面的客户端，我们选择使用命令行接口
- **Shell 集成**: 将 Git 状态集成到您的 shell 中会非常方便。([zsh](https://github.com/olivierverdier/zsh-git-prompt), [bash](https://github.com/magicmonty/bash-git-prompt))。[Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh)这样的框架中一般以及集成了这一功能
- **编辑器集成**: 和上面一条类似，将 Git 集成到编辑器中好处多多。[fugitive.vim](https://github.com/tpope/vim-fugitive) 是 Vim 中集成 GIt 的常用插件
- **工作流**: 我们已经讲解了数据模型与一些基础命令，但还没讨论到进行大型项目时的一些惯例 (
有[很多](https://nvie.com/posts/a-successful-git-branching-model/)
[不同的](https://www.endoflineblog.com/gitflow-considered-harmful)
[处理方法](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow))
- **GitHub**: Git 并不等同于 GitHub。在 GitHub 中您需要使用一个被称作[拉取请求（pull request）](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)的方法来向其他项目贡献代码
- **其他 Git 提供商**: GitHub 并不是唯一的。还有像 [GitLab](https://about.gitlab.com/) 和 [BitBucket](https://bitbucket.org/) 这样的平台。

## 资源

- [Pro Git](https://git-scm.com/book/en/v2) ，**强烈推荐**！学习前五章的内容可以教会您流畅使用 Git 的绝大多数技巧，因为您已经理解了 Git 的数据模型。后面的章节提供了很多有趣的高级主题。（[Pro Git 中文版](https://git-scm.com/book/zh/v2)）；
- [Oh Shit, Git!?!](https://ohshitgit.com/) ，简短的介绍了如何从 Git 错误中恢复；
- [Git for Computer Scientists](https://eagain.net/articles/git-for-computer-scientists/) ，简短的介绍了 Git 的数据模型，与本文相比包含较少量的伪代码以及大量的精美图片；
- [Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/)详细的介绍了 Git 的实现细节，而不仅仅局限于数据模型。好奇的同学可以看看；
- [How to explain git in simple words](https://smusamashah.github.io/blog/2017/10/14/explain-git-in-simple-words)；
- [Learn Git Branching](https://learngitbranching.js.org/) 通过基于浏览器的游戏来学习 Git；