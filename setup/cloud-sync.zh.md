---
title: "Mac 与 win 之间的云同步处理"
date: 2024-01-15T16:02:28+08:00
lastmod: 2024-01-15T16:02:28+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["git"]
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
## iCloud

iCloud 是 Mac 的默认云盘，Apple 为 Mac 提供了一定程度上的 iCloud 集成，在 `Apple ID -> iCloud -> iCloud 云盘` 中，可以选择让“桌面与文稿文件夹”也使用 iCloud 云盘，此时，你原先的 `/Users/Documents` 和 `/Users/Desktop` 会被替换为 `iCloud/Documents` 中的 `文稿 - zwy - mbp14` 以及 `icloud/Desktop` 中的 `桌面 - zwy - mpb14` 目录，同时 iCloud 中的 `文稿` 目录与 `桌面` 目录会被分别作为 Mac 的 `/Users/Documents` 与 `/Users/Desktop` 目录，推测是通过符号链接来实现的？

此后所有的对 `/Users/Docuements` 与 `/Users/Desktop` 的修改都会反映到 iCloud 中。

要注意一点，推荐关闭 iCloud 的 `优化 Mac 储存空间` 功能，否则当你的 Mac 存储空间不足的时候，`/Users/Documents` 与 `/Users/Desktop` 目录中的内容会变成仅保存在 iCloud 云端，而本地没有保存内容，这会导致在这两个目录中执行 `git` 与 `cd` 等命令非常卡顿。

![4pc5GT](https://pic-upyun.zwyyy456.tech/uPic/4pc5GT.png)

## OneDrive

Windows 也提供了对 OneDrive 的集成，但是开启之后，不像 iCloud 那么彻底，原先的 `C:\Users\zwyyy\Documents` 与 `C:\Users\zwyyy\Desktop` 目录依旧存在，只是文件管理器左侧显示的 `桌面` 与 `文档` 会指向 `C:\Users\zwyyy\OneDrive\Desktop` 和 `C:\Users\zwyyy\OneDrive\Documents` 两个目录，同时 Windows 的桌面显示的内容也被替换成了 `C:\Users\zwyyy\OneDrive\Desktop` 中的内容。

## 最佳实践

通过 **git 进行版本管理** 的代码文件不适合存放在云盘会自动同步的目录中，哪怕是使用 mega 这种自定义排除同步功能十分丰富的云盘，也不推荐，这是因为云盘对文件的同步容易被 git 视为修改，从而在 git pull 时发生 conflict，哪怕这个文件的内容你实际上没有修改。

推荐将 code 目录直接存放在 `/Users/zwyyy/` 目录下（Mac）或者 `C:\Users\zwyyy` 目录下（Windows）。

代码文件统一存放在 code 目录下，blog 也存放在 code 目录下。

> 源代码编译时会产生大量的临时文件，这也是不推荐将代码源文件通过云同步工具进行同步的一个重要原因。

## 代码文件的自动同步

代码文件一般通过 git 进行版本管理，因此不适合存放在云盘中进行同步，然而总有代码写了一部分，忘记 push 到 github，而回家之后，要接着修改代码的情况，那应该如何实现代码的自动同步呢？

对 Mac 或者 Linux，解决方案为通过 crontab 自动执行 `git push` 命令，具体一点，即定时执行以下命令：

```sh
git add -A
git commit -m "mac, auto sync"
git push ali main:mac # Linux 替换为 linux 分支，Windows 替换为 win 分支
```

Windows 应该差不多，即要求定时执行 powershell 脚本。

```sh
git add -A
git commit -m "win, auto push"
git push ali main:win
```

之后，假设回到家，发现之前忘记手动 push 了，那么只需要执行以下命令：

```sh
git checkout mac # 切换到 mac 分支
git pull gitlab mac
git checkout main
git merge mac # 将本地的 mac 分支的更改合并到 main 分支中

git push origin main
```

对于 windows，则是：

```sh
git checkout win
git pull ali win
git checkout main
git merge win
```

根据当前作息，我将设置 mac 在 18:10 与 20:40 定时执行 push。

### mac 设置 crontab

mac 可以通过 crontab 工具实现定时执行 push 操作。为了保证 crontab 可以正常执行，我们需要做一点额外配置。

首先执行 `crontab -e`，修改为如下内容并退出保存：

```sh
6 18 * * * zsh /Users/zwyyy/.config/mac-push.sh
7 18 * * * zsh /Users/zwyyy/code/bean/mac-push.sh
8 18 * * * zsh /Users/zwyyy/code/leetcode/mac-push.sh
9 18 * * * zsh /Users/zwyyy/code/blog/zwyb_blog/mac-push.sh

36 20 * * * zsh /Users/zwyyy/.config/mac-push.sh
37 20 * * * zsh /Users/zwyyy/code/bean/mac-push.sh
38 20 * * * zsh /Users/zwyyy/code/leetcode/mac-push.sh
39 20 * * * zsh /Users/zwyyy/code/blog/zwyb_blog/mac-push.sh
```

然后执行 `sudo launchctl list | grep cron` 查看任务是否存在，我的输出如下：

```sh
zwyyy@zwy-mbp14 ~/c/bean (main)> sudo launchctl list | grep cron                                                                                                                          (base)
458     0       com.vix.cron
```

有输出说明任务存在。然后执行 `locate com.vix.cron` 查看启动项配置，如果有 `WARNING`，就执行系统提供的命令 `sudo launchctl load -W /System/Library/LaunchDaemons/com.apple.locate.plist`。

然后查看 `/etc/crontab` 是否存在，我这里是不存在的，需要执行 `sudo touch /etc/crontab` 来创建。

之后，可能需要给 crontab 完全磁盘访问权限，crontab 安装于 `/usr/bin`，我们打开 `系统偏好设置 -> 隐私与安全性 -> 完全磁盘访问权限`，将访达中的 crontab 图标拖进去，即可为 crontab 启用完全磁盘访问权限。

但是我的 crontab 执行的命令似乎并不需要完全磁盘访问权限，一般来说，应用需要访问 User 的 Documents、Desktop、Downloads、OneDrive 这几个目录会需要单独授予磁盘访问权限，对 crontab，我们无法单独授予对某个文件夹的访问权限，因此只能授予完全磁盘访问权限。

> 关于 crontab 是否需要完全磁盘访问权限，这一点还有待测试。
> 当前测试表明不需要完全磁盘访问权限。

### Windows 设置定时任务

以 Win11 操作系统为例，打开 `任务计划程序`，点击 `任务计划程序库`，然后点击右侧的 `创建任务`，在 `触发器` 标签处选择每日重复，在 `操作` 标签下，`程序` 选择 `pwsh.exe`，`参数` 设置为 `-ExecutionPolicy Bypass -File "C:\path\to\your\script.ps1"`。

其中 `-ExecutionPolicy Bypass` 表示 PowerShell 脚本的执行策略为 `Bypass`，即允许所有 PowerShell 脚本运行而不受任何限制，`-File` 之后则是要执行的脚本的具体路径。


