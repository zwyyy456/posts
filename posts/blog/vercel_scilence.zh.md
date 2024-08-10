---
title: "Vercel 部署 Hugo"
date: 2022-11-16T16:46:59+08:00
lastmod: 2022-11-16T16:46:59+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["tips", "geek"]
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
## 让 Vercel 部署 GitHub 项目成功时，不再自动发送邮件通知

在`github`项目根目录下新建`vercel.json`，内容为:
```json
{
  "github": {
    "silent": true
  }
}
```

## Vercel 环境变量设置

Vercel 默认的 Hugo 版本可能很低，需要通过环境变量指定 Hugo 版本，如下：

![CXCdmH](https://pic-upyun.zwyyy456.tech/uPic/CXCdmH.png)

