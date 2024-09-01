---
title: "RemNote 入门教程"
date: 2023-12-14T14:57:19+08:00
lastmod: 2023-12-14T14:57:19+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["remnote", "trick"]
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
## 自定义字体

RemNote 的默认字体用于代码块时，显示效果十分拉垮，`i` 与 `e` 这两个字母的看起来明显比 `m` 之类的字母更细。

RemNote 可以通过 Custom CSS 来自定义显示效果，关键在于找到对应的 CSS selector。

我们可以在 Firefox 上打开 `www.remnote.com`，点开一篇笔记，例如 `200 秋招`，然后鼠标划词选中内容，右键点击 `检查`，如下图所示：

![sKepHFND5WZjm17](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065638.png)

因此，可以注意到，行内代码对应的 selector 为 `.rn-code-node .w-full .font-mono`，因此 Custom CSS 添加

```css
.rn-code-node .w-full, .font-mono {
    font-family: Monaco Nerd Font Mono;
    font-size: 14px
} 
```

而对于启用了 `Lab` 中的 `rem code block` 之后的代码块，使用同样的方式找到对应的 CSS selector：

![Mlti2HURrfZwqsF](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065642.png)

因此 Custom CSS 添加

```css
.rn-code-node .w-full, .font-mono {
    font-family: Monaco Nerd Font Mono;
    font-size: 14px
} 
```

同时，对于 RemNote 的笔记中的字体，这里修改为 `霞鹜文楷`，很好看。

```css
.rn-editor__rem__body__text {
    font-family: "LXGW WenKai", "霞鹜文楷";
}
```

mac 可以通过执行 `brew install font-lxgw-wenkai --cask` 来安装字体，系统中显示的字体名称为 `霞鹜文楷`，而 win 可以通过执行 `scoop install LXGWWenKai` 来安装，系统中显示的字体名称为 `LXGW WenKai`。

> 代码块与行内代码，实际上是我在 QQ 群里问到了 selector 之后，又用 Firefox 的开发者工具去验证这个 selector 而已。

## 筛选所有的 `Todo` rem

macOS 下按下 `cmd + p`，然后输入 `Todo`，创建名为 `Todo` 的 rem，在这个 rem 中即可看到所有的 `/todo` rem。
