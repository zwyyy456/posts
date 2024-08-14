---
title: "计算机教育缺失的一课：命令行环境"
date: 2024-06-15T15:18:13+08:00
lastmod: 2024-06-15T15:18:13+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["mit"]
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
## 任务控制

### 信号与终止进程

shell 会使用 unix 提供的信号机制来进行进程之间的通信，当一个进程接收到信号时，它会停止执行原来的任务、处理该信号、并基于该信号传递的信息来改变任务的执行，可以认为信号是一种 *软件中断*。

下面这个 python 程序演示了捕获 `SIGINT` 信号并忽略该信号，即这个程序在收到 `SIGINT` 信号时，不会终止程序，我们需要使用 `SIGQUIT` 信号来停止这个程序，可以通过 `<C-\>` 来发送该信号。

```py
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

运行该程序，向该程序发送两次 `SIGINT`，然后发送一次 `SIGQUIT`，程序反应如下：

```txt
zwyyy in 🌐 d3855u in ~/missing-semester 13s
 ❯ python3 sig.py
28^C
I got a SIGINT, but I am not stopping
53^C
I got a SIGINT, but I am not stopping
63^\zsh: quit       python3 sig.py
```

> 注意 `^` 是我们在终端输入 `Ctrl` 时的表现形式。

### 暂停与后台执行进程

信号除了终止进程之外，也可以让进程做其他事情，例如 `SIGSTOP` 会让进程暂停，在终端中，键入 `<C-z>` 会让 shell 发送 `SIGTSTP` 信号（`terminal` 版本的 SIGSTOP）。

我们可以使用 `fg` 或者 `bg` 来恢复暂停的工作，它们分别表示在前台继续或者在后台继续。

`jobs` 命令会列出当前终端会话中尚未完成的全部任务，可以使用 `pid` 来引用这些任务（也可以使用 `pgrep` 来找出 pid）。一种更符合直觉的操作是使用百分号+任务编号（`jobs` 会打印出任务编号）来选取该任务，可以用 `$!` 来选择最近一个任务。

在命令中添加 `&` 后缀可以让命令直接在后台运行，这样就能在 shell 中继续执行其他操作，但是该命令仍会使用 shell 的标准输出，例如 `sleep 1000 &`。

对于正在运行的前台进程，我们输入 `<C-z>`，该进程就会被暂停，再输入 `bg %<任务编号>` 就能让该进程在后台运行。要注意的是，后台运行的进程仍然是当前 shell 的子进程，一旦关闭当前终端（会发送 `SIGHUP` 信号），这些后台的进程也会被终止，可以使用 `nohup` 运行进程来防止这种情况（忽略 `SIGHUP`）。

以下这个简单的会话中展示了上面这些概念的应用：

```txt
$ sleep 1000
^Z
[1]  + 18653 suspended  sleep 1000

$ nohup sleep 2000 &
[2] 18745
appending output to nohup.out

$ jobs
[1]  + suspended  sleep 1000
[2]  - running    nohup sleep 2000

$ bg %1
[1]  - 18653 continued  sleep 1000

$ jobs
[1]  - running    sleep 1000
[2]  + running    nohup sleep 2000

$ kill -STOP %1
[1]  + 18653 suspended (signal)  sleep 1000

$ jobs
[1]  + suspended (signal)  sleep 1000
[2]  - running    nohup sleep 2000

$ kill -SIGHUP %1
[1]  + 18653 hangup     sleep 1000

$ jobs
[2]  + running    nohup sleep 2000

$ kill -SIGHUP %2

$ jobs
[2]  + running    nohup sleep 2000

$ kill %2
[2]  + 18745 terminated  nohup sleep 2000

$ jobs
```

`SIGKILL` 是一个特殊的信号，它不能被进程捕获并且它会马上结束该进程。不过这样做会有一些副作用，例如留下孤儿进程

## 终端复用器：tmux

[`tmux`](https://www.man7.org/linux/man-pages/man1/tmux.1.html) 这类的终端多路复用器可以允许我们基于 panes 和 tabs 分割出多个终端窗口（即打开一个 terminal 窗口相当于打开了多个），这样您便可以同时与多个 shell 会话进行交互。

tmux 也允许我们分离当前 session 并在将来重新连接，被分离的 session 内的 shell 不会收到 `SIGHUP` 信号。

tmux 有三个核心概念：

- sessions
    - windows
        - panes

这三个概念的层级如上所示。windows 可以理解为浏览器中的 tabs。

`tmux` 的快捷键需要我们掌握，它们都是类似 `<C-b> x` 这样的组合，即需要先按下`Ctrl+b`，松开后再按下 `x`。`tmux` 中对象的继承结构如下：

- **会话** - 每个会话都是一个独立的工作区，其中包含一个或多个窗口
    + `tmux` 开始一个新的会话
    + `tmux new -s NAME` 以指定名称开始一个新的会话
    + `tmux ls` 列出当前所有会话
    + 在 `tmux` 中输入 `<C-b> d`  ，将当前会话分离
    + `tmux a` 重新连接最后一个会话。您也可以通过 `-t` 来指定具体的会话


- **窗口** - 相当于编辑器或是浏览器中的标签页，从视觉上将一个会话分割为多个部分
    + `<C-b> c` 创建一个新的窗口，使用 `<C-d>`关闭
    + `<C-b> N` 跳转到第  _N_ 个窗口，注意每个窗口都是有编号的
    + `<C-b> p` 切换到前一个窗口
    + `<C-b> n` 切换到下一个窗口
    + `<C-b> ,`  重命名当前窗口
    + `<C-b> w` 列出当前所有窗口

- **面板** - 像 vim 中的分屏一样，面板使我们可以在一个屏幕里显示多个 shell
    + `<C-b> "` 水平分割
    + `<C-b> %` 垂直分割
    + `<C-b> <方向>` 切换到指定方向的面板，<方向> 指的是键盘上的方向键
    + `<C-b> z` 切换当前面板的缩放
    + `<C-b> [` 开始往回卷动屏幕。您可以按下空格键来开始选择，回车键复制选中的部分
    + `<C-b> <空格>` 在不同的面板排布间切换

扩展阅读：
[这里](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) 是一份 `tmux` 快速入门教程。

## 别名与配置文件

### 别名

当我们在 shell 中执行 `alias ll="ls -lh"` 后，执行 `ll` 命令就相当于执行 `ls -lh`。

```txt
zwyyy in 🌐 d3855u in ~/missing-semester
 ❯ ls
nohup.out  sig.py  words.txt

zwyyy in 🌐 d3855u in ~/missing-semester
 ❯ ls -lh
total 968K
-rw------- 1 zwyyy zwyyy    0 Jun 15 15:29 nohup.out
-rw-r--r-- 1 zwyyy zwyyy  246 Jun 15 15:39 sig.py
-rw-r--r-- 1 zwyyy zwyyy 962K Jun 15 14:55 words.txt

zwyyy in 🌐 d3855u in ~/missing-semester
 ❯ alias ll="ls -lh"

zwyyy in 🌐 d3855u in ~/missing-semester
 ❯ ll
total 968K
-rw------- 1 zwyyy zwyyy    0 Jun 15 15:29 nohup.out
-rw-r--r-- 1 zwyyy zwyyy  246 Jun 15 15:39 sig.py
-rw-r--r-- 1 zwyyy zwyyy 962K Jun 15 14:55 words.txt
```

当我们关闭这个 shell 或者切换到另一个 shell 后，别名就不再生效，为了让别名对所有 shell 生效，需要将它写入 shell 的配置文件中，以我这里使用的 zsh 为例，在 `~/.zshrc` 中追加 `alias ll="ls -lh"`，再执行 `source ~/.zshrc` 后，别名就将一直生效。

### 配置文件

很多程序的配置都是通过纯文本格式的被称作 *dotfile* 的配置文件来完成的（之所以称为 dotfile，是因为它们的文件名以 `.` 开头，例如 `~/.vimrc`。也正因为此，它们默认是隐藏文件，`ls`并不会显示它们）。

shell 的配置也是通过这类文件完成的。在启动时，您的 shell 程序会读取很多文件以加载其配置项。根据 shell 本身的不同，您从登录开始还是以交互的方式完成这一过程可能会有很大的不同。关于这一话题，[这里](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html) 有非常好的资源

对于 `bash`来说，在大多数系统下，您可以通过编辑 `.bashrc` 或 `.bash_profile` 来进行配置。在文件中您可以添加需要在启动时执行的命令，例如上文我们讲到过的别名，或者是您的环境变量。

实际上，很多程序都要求您在 shell 的配置文件中包含一行类似 `export PATH="$PATH:/path/to/program/bin"` 的命令，这样才能确保这些程序能够被 shell 找到。

还有一些其他的工具也可以通过 *dotfile* 进行配置：

- `bash` - `~/.bashrc`, `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` 和  `~/.vim` 目录
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

此外，也有一些工具，其配置文件位于 `~/.config/xxx/` 目录下：

- `fish` - `~/.config/fish/config.fish`
- `gdb` - `~/.config/gdb/gdbinit`
- `lazygit` - `~/.config/lazygit/config.yaml`


我们应该如何管理这些配置文件呢，它们应该在它们的文件夹下（例如统一放在 `~/.config`），并使用版本控制系统进行管理，然后通过脚本将其 **符号链接** 到需要的地方。这么做有如下好处：

- **安装简单**: 如果您登录了一台新的设备，在这台设备上应用您的配置只需要几分钟的时间；
- **可移植性**: 您的工具在任何地方都以相同的配置工作
- **同步**: 在一处更新配置文件，可以同步到其他所有地方
- **变更追踪**: 您可能要在整个程序员生涯中持续维护这些配置文件，而对于长期项目而言，版本历史是非常重要的

## 连接远程设备

参见 [ssh 简单教程]()
