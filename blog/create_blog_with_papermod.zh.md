---
title: "基于 hugo 和 papermod 主题搭建自己的博客"
date: 2022-11-04T18:22:26+08:00
lastmod: 2022-11-04T18:22:26+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["geek", "hugo", "tips"]
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

## 部署博客到vercel
### FreeNom申请域名
首先，梯子最好选择美国的，并且freenom选择地址时最好与ip所在州可以对应得上；
进入[FreeNom](https://www.freenom.com/)，输入`zwyb.tk`，然后点击`检查可用性`，这里要记得输入后缀，能避免点击`现在获取`显示不可用的问题。
如下图所示:
![B2n6EKpycAxTe1D](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065730.png)


### Cloudfare管理域名
cloudfare添加站点`zwyyy456.ml`，然后添加DNS record，内容如下图所示：
![83b4GExRivpB2tV](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065732.png)

下一步，进入freenom, `Services->My Domains->Manage Domain->Management Tools->Nameservers`，选择`Use custom nameservers(enter blow)`，填入cloudfare生成的nameservers。注意cloudfare的SSL/TLS策略必须是`Full`。

### vercel部署博客和绑定域名
将整个项目，如`papermod`这个文件夹，作为一个git仓库上传到github，然后vercel创建新项目，选择`continue with github`，就能将对应的仓库导入到vercel，部署的时候注意添加`Environment Variables`
```
HUGO_VERSION 0.93.0
```

项目部署好之后，点击该项目，`Settings->Domains`，添加之前FreeNom申请的域名，DNS record在上一步cloudfare管理域名那里已经添加过了。
