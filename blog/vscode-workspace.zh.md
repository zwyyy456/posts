---
title: "VSCode 工作空间（Workspace）指北"
date: 2024-05-25T10:33:04+08:00
lastmod: 2024-05-25T10:33:04+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["vscode"]
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
## 为什么要用工作区

VSCode 作为一个轻量的编辑器，对比 IDE 很多功能并非集成。如果想打造成一个 IDE，那么就需要安装很多扩展，然而扩展越多，管理起来也越发困难，VSCode 也就越发“笨重”，例如，当我进行 `cpp` 开发的时候，`python` 与 `go` 的相关插件就不需要了，然而其默认也是开启的，我们当然可以手动关闭，有需要的时候再打开。然而，随着这样的扩展越来越多，手动控制扩展的开启与关闭就变得非常麻烦。

我认为 VSCode 的工作区（Work Space）诞生的一大重要原因就是扩展的管理。事实上，当我们打开 VSCode 的时候，其实就位于默认的工作区中。

## 工作区的创建

假设我只会在 `~/code/blog/zwyb_blog` 目录下用 VSCode 写博客，那么我用 VSCode 打开 `~/code/python` 目录，我就可以点击 `File` -> `Save Workspace as`，保存为 `blog.code-workspace`。

目前，其内容如下：

```json
{
	"folders": [
		{
			"path": "."
		},
	],
	"settings": {}
}
```

可以看到，内容分为了 `folders` 与 `settings` 两大类，`folders` 表示处于该工作空间的文件夹所在目录，我们可以点击 `File` -> `Add Folder to Workspace` 来添加其他项目文件夹目录到该工作区，例如我添加了 `~/code/blog/papermod` 目录，添加后内容修改为如下：

```json
{
	"folders": [
		{
			"path": "."
		},
		{
			"path": "../papermod"
		},
	],
	"settings": {}
}
```

遗憾的是，VSCode 中不支持在配置文件中直接设置扩展的激活与否，需要手动在 VSCode 的扩展视图中点击齿轮按钮，设置 `Disable(Workspace)`。

可以先直接禁用所有的扩展，然后按需选择是仅在工作空间启用还是所有工作空间都启用，如下图所示：

![4F48ly](https://pic-upyun.zwyyy456.tech/uPic/4F48ly.png)

![r4dXH9](https://pic-upyun.zwyyy456.tech/uPic/r4dXH9.png)

## 工作区的配置

VSCode 的配置文件的级别可以分为 `User`、`Workspace` 与 `Folder` 三个级别，其中 `Folder` 级别表示文件夹下的 `.vscode` 文件夹中的 `settings.json` 文件。

配置文件的优先级为 `Folder` > `Workspace` > `User`，对于一个配置项，如果在更高优先级的配置文件中被设置了，那么该配置项会被覆盖。

例如，假设 `~/code/blog/zwyb_blog` 目录下存在 `.vscode/settings.json` 文件，其内容为设置字体大小为 `14`，`blog.code-workspace` 中的 `settings` 部分设置字体大小为 `13`，用户配置文件中设置字体大小为 `12`，则打开 `blog.code-workspace` 工作区，打开 `zwyb_blog` 目录下的文件，字体大小显示为 `14`，对于非 `zwyb_blog` 目录下的文件，例如 `papermod` 目录下的文件，则字体大小为 `13`。

这里要注意的是，`blog.code-workspace` 中的设置内容，必须在你打开该工作空间时才会生效，例如 `~/code/blog/papermod` 虽然位于 `blog.code-workspace` 中，但是 VSCode 直接打开该文件夹，`blog.code-workspace` 中的设置不会生效，该文件夹中的文件的字体仍为 `12`。

此外，使用 VSCode Remote 也能打开远程机器上的工作空间，与本地机器的工作空间区别不大，具体效果留待读者探索了。