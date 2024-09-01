---
title: "miniconda 基础教程"
date: 2022-11-11T15:55:04+08:00
lastmod: 2022-11-11T15:55:04+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["python"]
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
## 创建 python 虚拟环境
安装 python 指定环境
`conda create -n zwyb python=3.9`

安装 python 指定环境的时候安装相应的包
`conda create -n zwyb python=3.9 pandas`

进入指定的环境
`conda activate zwyb`

退出当前环境
`conda deactive zwyb`

显示所有环境
`conda env list`

删除指定的环境
`conda env remove -n zwyb`

## 更换清华源
vim 编辑`~/.condarc`，将其中内容修改为
```
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch-lts: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

更换`pip`源，执行
`pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple`

## 关闭启动终端后自动进入 conda 环境
`conda config --set auto_activate_base false`


