---
title: "ssh简单教程"
date: 2023-03-14T16:23:37+08:00
lastmod: 2023-03-14T16:23:37+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["geek", "ssh"]
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
## ssh配置免密码登录服务器
### 生成密钥对
执行`ssh-keygen -t ed25519 -C "zwyyy456@hotmail.com"`以生成密钥对，存放在`~/.ssh`文件夹下，`id_ed25519.pub`为公钥，`id_ed25519`为私钥。

### 上传公钥到服务器
这里以我的N1为例，执行`ssh-copy-id -i ~/.ssh/id_ed25519.pub root@192.168.6.217`和`ssh-copy-id -i ~/.ssh/id_ed25519.pub zwyyy@192.168.6.217`，将公钥上传到服务器，`root`和`zwyyy`分别是两个用户。

## 配置局域网ssh连接到wsl
### hyper-v创建虚拟交换机
打开hyper-v管理器，选择**虚拟交换机管理器**，选择创建**外部**类型的虚拟交换机，这里命名为`wsl_ssh`。

### win11上新建wsl配置文件
```sh
cd ~
New-Item .wslconfig
nvim .wslconfig
```
修改配置文件内容为
```toml
[wsl2]
networkingMode=bridged
vmSwitch=wsl_ssh # 这里为你创建的虚拟交换机名称
ipv6=true
```
之后执行`wsl --shutdown`再启动`wsl`，就会发现`ip`地址为电脑本身的ip了。

### openwrt上固定电脑的ip
进入**openwrt**的管理界面，点击`网络->DHCP/DNS`，选择静态地址分配，固定windows笔记本的ip

### 启用wsl上的ssh
执行`sudo nvim /etc/ssh/sshd_config`，将`#port 22`修改为`port 2222`，取消注释`#PasswordAuthentication yes`和`#PubekyAuthentcation yes`，重启**ssh**服务，执行`sudo service ssh restart`。

### win11设置端口转发
参照该[链接](https://blog.csdn.net/lcuwb/article/details/82885920)

之后同一局域网的mac执行`ssh -p 2222 zwyyy456@192.168.6.209`，即可连接到**wsl**。
