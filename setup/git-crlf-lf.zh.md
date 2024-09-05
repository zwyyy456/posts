---
title: "Git 如何避免 'warning: LF will be replaced by CRLF'"
date: 2023-12-18T22:02:46+08:00
lastmod: 2023-12-18T22:02:46+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["git", "trick"]
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

前面提到，我会使用 Filen 来同步文件，例如当我在 macOS 上创建了 `.md` 文件而忘记 push 到 GitHub 时，回到宿舍，打开我 Windows 系统的 Acer 笔记本（后续以 Acer 指代该笔记本）时，Filen 会自动将 `.md` 文件同步到 Acer，然后当我在 Acer 上完成对该 `.md` 的编辑并 commit 时，就会提示如下内容：

> CR 表示 \r，LF 表示 \n

```txt
warning: LF will be replaced by CRLF in content/posts/blog/n1-plex-music.zh.md.
The file will have its original line endings in your working directory
```

我们在安装 Git 时，默认 `core.autocrlf = true`，即 Git 会认为，工作区的文件都应该用 `\r\n` 来换行，如果工作区因为新增（在这个情境下就是因为 Filen 把文件同步到了 Acer）或编辑出现了 `\n` 换行符的文件，`git add` 这些文件时，发现准备提交的文件是 `\n` 作为换行符，就会出现这个警告，并提示哪些文件是 `\n` 换行的，但是 Git 不会对工作区这些文件做换行符的转换）。

> 工作区（Working Directory）：这是您本地文件系统中的目录，包含了项目的实际文件。您可以在工作区中进行文件的创建、编辑和删除操作。

> 暂存区（Staging Area）：也称为索引，是一个中间区域，用于准备要提交到Git存储库的更改。您可以使用 `git add` 命令将工作区中的文件和更改添加到暂存区，然后使用git commit命令将它们提交到存储库。

> 存储库（Repository）：存储库是Git用来保存项目的历史记录和元数据的地方。它包含了所有以前提交的版本、分支信息、标签和提交记录。存储库通常位于项目目录的.git子目录中。

`core.autocrlf = true` 表示在 commit 时，将工作区文件的换行符由 `\r\n` 转换为 `\n`，即 Repository 的文件一定是 `\n` 换行的，而在 checkout 或者 pull 时，会将 checkout 或者 pull 到本地的文件转换为 `\r\n` 换行，这就是 [解决 macOS 上的 fish 出现的 'et_color Unkown color ' 问题](https://blog.zwyyy456.tech) 这篇文章碰到的问题的根本原因。

> 我先执行了 `git pull`，将 macOS 上对 `.config` 目录的修改同步到了本地，由于 `fish_variables` 在 macOS 上可能发生了修改，于是 `pull` 的时候，Windows 中的 `~/.config/fish/fish_variables` 在从 Github pull 到本地时，换行符就被从 `\n` 转换成了 `\r\n`，然后 `\r\n` 换行同步的文件就被 Filen 同步到了 macOS 上去了，macOS 上 fish 就无法正确识别 `fish_variables` 中定义的变量，就出现了 'et_color Unkown color ' 问题。

## 解决方案

首先，执行 `git config --global core.autocrlf input`，表示在 commit 时，将 `\r\n` 自动转换成 `\n`，而 pull 或者 checkout 时，不转换 `\r\n` 或者 `\n`，保持原样。

然后，我们需要修改 VsCode、NeoVim、Sublime Text 这三个编辑器的设置，让它们即使在 Windows 下也以 `\n` 作为换行。

对 VsCode，设置中搜索 `eol`，修改为 `\n`；对于 Sublime Text，修改 `View -> Line Endings` 为 `Unix`；对于 Neovim，在 `nvim/lua/config/options.lua` 中追加 `vim.opt.fileformat = 'unix'`。

> 我平时会用到的编辑器就是以上三个。

这个解决方案比较暴力，就是保证不管是 Windows 还是 macOS 都使用 `\n` 来进行换行。



