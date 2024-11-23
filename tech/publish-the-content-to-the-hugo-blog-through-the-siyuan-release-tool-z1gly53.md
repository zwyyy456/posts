---
title: 通过思源发布工具将内容发布到 Hugo 博客
slug: publish-the-content-to-the-hugo-blog-through-the-siyuan-release-tool-z1gly53
url: >-
  /post/publish-the-content-to-the-hugo-blog-through-the-siyuan-release-tool-z1gly53.html
date: '2024-11-21 23:32:04+08:00'
lastmod: '2024-11-23 17:25:58+08:00'
toc: true
isCJKLanguage: true
---

# 通过思源发布工具将内容发布到 Hugo 博客

## 前期准备

首先，安装好思源笔记与思源发布工具插件，然后点击 `发布工具->通用设置->发布设置`​，在插件商店中，选择 `Github`​ 下的 `Hugo`​，之后会弹出一个 `添加 Hugo 平台`​ 的页面，点击提交即可。然后点击 `发布工具->通用设置->发布设置`​ 下的发布配置，能看到会有一个已启用的 Hugo 的配置项，点击旁边的设置按钮，就进入了如下的设置页面：

其中，仓库名是你的 GitHub 中的对应仓库的仓库名，我这里是 `posts`​，作为 submodule 添加到我的 hugo 博客的仓库中，`存储目录`​ 这里其实不太关键，只有在点击一键发布的时候，才会被用到，我这里会默认选择 `tech`​，与个人设置有关，如果是发布到 Hugo 博客，那么这个发布工具做的实质上就是将本篇文章转换成 `md`​ 文件，添加到 GitHub 的对应的仓库的对应目录下，`文件规则`​ 对应的是生成文件名的规则。

​![jVWdCm](https://pic-upyun.zwyyy456.tech/uPic/jVWdCm.png)​

文章预览规则和预览规则可以先不管，暂时保持默认即可，发布目录也保持 `tech`​，这里还没明确发布目录和存储目录各自的作用，

​![H1pimo](https://pic-upyun.zwyyy456.tech/uPic/H1pimo.png)​

​​

​![image](https://pic-upyun.zwyyy456.tech/siyuan20241121233718.pngnull)​
