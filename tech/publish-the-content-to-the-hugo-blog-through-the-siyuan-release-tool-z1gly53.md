---
title: 通过思源发布工具将内容发布到 Hugo
authors:
  - zwyyy456
categories:
  - setup
tags:
  - hugo
  - siyuan
description: ''
weight: null
slug: publish-siyuan-article-to-hugo.zh
draft: false
comments: true
showToc: true
TocOpen: true
hidemeta: false
disableShare: true
showbreadcrumbs: false
---



## 前期准备

首先，安装好思源笔记与思源发布工具插件，然后点击 `发布工具->通用设置->发布设置`​，在插件商店中，选择 `Github`​ 下的 `Hugo`​，之后会弹出一个 `添加 Hugo 平台`​ 的页面，点击提交即可。然后点击 `发布工具->通用设置->发布设置`​ 下的发布配置，能看到会有一个已启用的 Hugo 的配置项，点击旁边的设置按钮，就进入了如下的设置页面：

其中，仓库名是你的 GitHub 中的对应仓库的仓库名，我这里是 `posts`​，作为 submodule 添加到我的 hugo 博客的仓库中，`存储目录`​ 这里其实不太关键，只有在点击一键发布的时候，才会被用到，我这里会默认选择 `tech`​，与个人设置有关，如果是发布到 Hugo 博客，那么这个发布工具做的实质上就是将本篇文章转换成 `md`​ 文件，添加到 GitHub 的对应的仓库的对应目录下，`文件规则`​ 对应的是生成文件名的规则。

​![jVWdCm](https://pic-upyun.zwyyy456.tech/uPic/jVWdCm.png)​

文章预览规则和预览规则可以先不管，暂时保持默认即可，发布目录也保持 `tech`​，这里还没明确发布目录和存储目录各自的作用，

> 根据测试结果，`.md`​ 文件在选择一键发布时，会被直接发布到 `存储目录`​ 对应的目录，而 `发布目录`​ 默认与存储目录保持一致，这里是一个无效的选项。

鉴权 `token`​ 是点击 `token`​ 生成地址去生成的 api token，点击生成 token 的时候选择 `classic`​ 的那一个（这样可以选择 token 的有效期为永久，权限只要把 repo 那一栏都勾选上即可。

​![H1pimo](https://pic-upyun.zwyyy456.tech/uPic/H1pimo.png)​

然后在发布工具的偏好设置中，勾选 `去除正文 h1`​，否则生成的文章中，会生成一个文章名对应的一级标题。

点击发布工具的一键发布后，文章对应的 `md`​ 会被上传到对应的目录中，这时，我们在思源中，查看该文件的 `更多 -> 属性`​，在自定义中，可以看到有一个 `github-Hugo-yaml`​ 文件，内容如下，插件在执行一键发布时，会自动在 `.md`​ 文件的内容头部添加以下内容，这里的 `slug`​ 与前面的文件规则中的 `[slug].md`​ 是对应的，即生成的 `.md`​ 文件的文件名是 `publish-xxx.md`​ 。

​![N6KWn9](https://pic-upyun.zwyyy456.tech/uPic/N6KWn9.png)​

事实上，点击发布工具的 `常规发布`​，点击我们的 `Hugo`​ 项，在 `详细发布`​ 中，我们可以选择要将文件发布到哪个目录，在 `源码模式`​，我们可以修改 `yaml`​ 标记的内容，还可以修改 `.md`​ 文件的内容，而 `同步修改到思源笔记`​ 按钮可以将我们的修改同步到思源笔记的文章中。我常用的 `yaml`​ 标记的模板内容如下：

```yaml
title: ""
authors:
  - "zwyyy456" # 作者
categories:
  - "setup" # or tech
tags:
  - ""
description: "" # 描述
weight: null # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: x.zh
draft: false # 是否为草稿
comments: true # 是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false # 顶部显示当前路径
```

因此，这里需要我修改的 yml 中，其实只有 `date`​ 与 `lastmod`​、`title`​ 这三个字段不变，其他都还是要修改。

该插件似乎还可以将仓库中已经存在的 `.md`​ 文件与思源笔记中的文章对应起来，该功能留待以后探索吧。

‍
