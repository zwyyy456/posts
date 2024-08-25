---
title: "解决远程主机的默认 shell 为 fish 时，vscode remote 无法连接的问题"
date: 2023-07-08T11:30:15+08:00
lastmod: 2023-07-08T11:30:15+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["tips"]
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
## 问题描述

我主要用的 shell 就是 fish，主打一个开箱即用，虽然也配置过 zsh，但是感觉配置好的 zsh 在易用性上也就是 fish 的水平。

此前，一直以来默认的 shell 都是 bash，ssh 或者 vscode remote 远程连接上去之后，再输入 `fish` 来进行手动切换，后来嫌麻烦，就执行 `chsh -s /usr/bin/fish` 将默认 shell 切换到了 fish，然后 vscode remote 就连接不上了。

出现该问题的原因见该 [issue](https://github.com/microsoft/vscode-remote-release/issues/2509) 里的讨论

> we still have a bug connecting to remotes with fish shells as their default shell. Using the `remotePlatform` setting we added a work around to make the connection work. It's not ideal but it works. The bug specifically has to do with what seems that Fish shells don't let us pipe in scripts after connection unless we connect with a command like `ssh your_host bash`. However, we want to avoid using "bash" since it prevents usage of RemoteCommand. For now it's not clear what the best way to accommodate both RemoteCommands and Fish shells. The `remotePlatform` setting gives me a workaround where if I see you have that set I add "bash" to the connection command and disable RemoteCommand.

## 解决方案

在上面提到的 [issue](https://github.com/microsoft/vscode-remote-release/issues/2509) 里面有给出解决方案，即打开设置，搜索 `remote.SSH.remotePlatfrom`，选择 `add Item`，添加键值对，key 是远程主机的名字，例如 `deb-lenovo`，value 则是平台，选择 linux，即可。

这样做其实还是有一个副作用，那就是 vscode remote 连接上去之后，new Terminal 的默认 shell 还是 bash，所以说这个解决方案并不优雅。
