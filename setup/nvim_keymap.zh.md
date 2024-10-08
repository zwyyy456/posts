---
title: "Neovim 快捷键配置"
date: 2023-11-05T17:05:51+08:00
lastmod: 2023-11-05T17:05:51+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["neovim", "geek", "tips"]
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
## Vim 快捷键

这一部分是 Vim 的默认快捷键，例如 `gg`、`dd` 等。

### normal 模式快捷键

| 按键      | 操作                   | 按键      | 操作                    |
|-----------|------------------------|-----------|-------------------------|
| `i`       | 切换到插入模式         | `:`       | 切换到命令模式          |
| `h`       | 左移一个字符           | `j`       | 下移一个字符            |
| `k`       | 上移一个字符           | `l`       | 右移一个字符            |
| `0`       | 移至行首               | `$`       | 移至行尾                |
| `^`       | 移至本行第一个非空字符 | `w`       | 向右移动一个单词        |
| `W`       | 向右移动一个单词（以空格分隔） | `2w`  | 向右移动两个单词        |
| `2W`      | 向右移动两个单词（以空格分隔） | `b`  | 向左移动一个单词        |
| `B`       | 向左移动一个单词（以空格分隔） | `2b`  | 向左移动 2 个单词       |
| `2B`      | 向左移动 2 个单词（以空格分隔） | `G`  | 移至文档末尾            |
| `gg`      | 移至文档首行           | `a`       | 光标后插入              |
| `A`       | 移至行末插入           | `o`       | 光标下插入一行          |
| `O`       | 光标上插入一行         | `x`       | 删除光标处字符          |
| `dw`      | 删除一个词             | `d0`      | 删至行首                |
| `d$`      | 删至行尾               | `d)`      | 删至句末                |
| `dgg`     | 删至文件开头           | `dG`      | 删至文件末尾            |
| `dd`      | 删除该行               | `2dd`     | 删除两行                |
| `r`       | 替换当前字符           | `R`       | 切换到 REPLACE 模式     |
| `u`       | 撤回操作               | `<C-r>`   | 重做撤回的操作          |
| `yy`      | 复制当前行             | `p`       | 在当前行之后粘贴内容    |
| `P`       | 在当前行之前粘贴内容   | `v`       | 打开 VISUAL 模式菜单    |
| `V`       | 切换到逐行选择的 VISUAL 模式 | `/` | 向后搜索                 |
| `?`       | 向前搜索               | `n`       | 下一个搜索结果          |
| `N`       | 上一个搜索结果         | `g`       | goto 菜单               |
| `gd`      | 跳转到定义             | `gcc`     | 注释当前行                |
| `<C-w>`   | window 菜单            | `<C-w>s`  | 水平拆分窗口            |
| `<C-w>v`  | 垂直拆分窗口           | `<C-q>`   | V-BLOCK 模式            |

其中，`<C-w>s` 表示同时按下 `Ctrl` 和 `w` 键，再马上按下 `s` 键。

### VISUAL 模式下的快捷键

| 按键      | 操作                   | 按键      | 操作                    |
|-----------|------------------------|-----------|-------------------------|
| `~`       | 切换大小写         | `d`       | 删除          |
| `c`       | 变更          | `y`       | 拷贝 |
| `>` | 增加锁进 | `<` | 减少缩进 |

## Neovim 的快捷键

这一部分是 neovim 使用 LazyVim 配置之后设置的一些快捷键。

> 当然，原始状态的 neovim 可能就有这些快捷键了。

注意，打开 neovim 之后，我们可以通过 `:` 进入命令模式后，可以利用 `cd /x/xx` 来移动到对应的目录。

> 注意，此后 `:xx` 即表示在命令模式下输入 `xx`。

`:checkhealth` 命令用于检测 Neovim 的工作状态，可以根据结果提示来安装缺少的组件。

`:Mason` 是由 mason.nvim 插件提供的命令，用于呼出 mason.nvim 的管理界面，可以用来管理 [LSP](https://microsoft.github.io/language-server-protocol/) 插件，如下图所示：

![uMJnOy](https://pic-upyun.zwyyy456.tech/uPic/uMJnOy.png)

通过 `<leader>e` 或者 `<leader>E` 可以在侧边栏打开文件浏览窗口，该窗口的操作快捷键如下：

| 按键  | 操作               | 按键  | 操作       |
|-------|--------------------|-------|------------|
| `j`   | 下移               | `k`   | 上移       |
| `<cr>`| 打开               | `a`   | 新建文件或目录 |
| `A`   | 新建目录           | `d`   | 删除       |
| `r`   | 重命名             | `c`   | 复制       |
| `m`   | 移动               | `q`   | 关闭窗口   |


> `<leader>e` 是打开从终端打开 nevoim 时所处目录的文件浏览窗口，`<leader>E` 则是当前文件所处目录的文件浏览窗口。

通过 `:te`，我们可以在 Neovim 新建一个 buffer（这里的 buffer 可以理解为标签页），这个新的 buffer 中则是一个命令行标签页，如下图所示：

![TlVIp8](https://pic-upyun.zwyyy456.tech/uPic/TlVIp8.png)

我们可以通过 `:bn` 切换到下一个 buffer，通过 `:bp` 切换到上一个 buffer。

通过 `<leader>b`，我们会进入到关于 buffer 的快捷键操作界面，如下图所示：

![4iSYdX](https://pic-upyun.zwyyy456.tech/uPic/4iSYdX.png)

当我们通过 `:te` 打开一个终端时，我们默认是处于 `Normal` 模式的，按下 `i` 则会切换到 `Terminal` 模式，然后就能像正常终端那样输入命令了，如果要切换回 `Normal` 模式，可以连按两次 `<esc>` 或者按下 `<C-\>` 再按下 `<C-n>`。

我们可以通过 `<C-w>` 或者 `<leader>w` 来进入窗口的快捷键操作界面，例如，我们可以通过 `<C-w>v` 可以将 Neovim 分成左右两个窗口，此时如果再输入 `:te`，那么原先的编辑器和终端则会默认一个占据左边的窗口，一个占据右边的窗口，如下图所示。

![E9sLSE](https://pic-upyun.zwyyy456.tech/uPic/E9sLSE.png)
