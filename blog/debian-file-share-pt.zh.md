---
title: "debian 配置文件共享（samba、nfs）与 pt"
date: 2023-07-21T18:03:01+08:00
lastmod: 2023-07-21T18:03:01+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: [""]
tags: [""]
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
## nfs

首先执行 `sudo apt install nfs-kernel-server` 安装 nfs，然后执行 `sudo nvim /etc/exports` 编辑 `/etc/exports` 文件，添加以下内容

```sh
# share documents
/home/zwyyy/Documents 192.168.6.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)

# share downloads
/home/zwyyy/Downloads 192.168.6.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)

# external disk
/home/zwyyy/mnt/ext 192.168.6.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
```

最前面是路径，根据自己需求来写。

192.168.6.0/24 是为了保证同一局域网下都能连接，all_squash,anonuid,anongid 保证客户端连接到 nfs 服务端的时候，都是以 uid=1000,gid=1000 用户和用户组连接的，这样就解决了 mac 连接上 nfs 之后无法写入的问题。

然后 debian 执行以下命令
```sh
sudo exportfs -ra
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```

然后 mac 执行 `sudo mount -t nfs -o resvport 192.168.6.155:/home/zwyyy/Documents ~/Downloads/deb-doc` 将 nfs 服务端的路径挂载到本地，具体路径根据自己需求来写。

> 注意确保执行挂载命令前，mac 的对应目录已经存在。

> Windows 不建议通过 nfs 方式连接到 nas。

## samba

首先执行 `sudo apt install samba` 安装 samba 服务，执行 `sudo nvim /etc/samba/smb.conf`，编辑 Share Definitions 部分：

```txt
[homes]
    comment = Home Directories
    path = /home/zwyyy
    browseable = yes
    read only = no
    writable = yes
    # 当 Windows 通过 samba 连接到 Debian 时，windows 通过 samba 在 Debian 创建文件时的文件权限为 0764
    create mask = 0764
    directory mask = 0764
    valid users = %s
```

> `path` 修改为要共享的路径。

然后执行 `sudo smbpasswd -a zwyyy` 创建对应的 samba 用户 zwyyy，这个过程会需要你输入密码，之后便可以通过这个用户连接到 samba 了，注意创建的 `samba` 用户名必须是 Debian 系统中已经存在的用户名。

然后执行 `sudo systemctl restart smbd`，之后就能在 Windows 上通过 samba 连接到 Debian 了。

打开 Windows 的文件管理，右键点击 `此电脑`，点击 `映射网络驱动器`，盘符随便选，网址为 `\\192.168.6.181\homes`，注意 ip 改为自己要连接的设备的 ip，`homes` 改为与上面 `Share Definitions` 中方括号包裹的内容，如下图所示：

![](https://pic-upyun.zwyyy456.tech/picgo/20240616120056.png)

![](https://pic-upyun.zwyyy456.tech/picgo/20240616120135.png)




