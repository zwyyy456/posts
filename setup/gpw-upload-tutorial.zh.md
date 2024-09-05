---
title: "简单的 GPW 发种教程"
date: 2023-12-31T15:38:45+08:00
lastmod: 2023-12-31T15:38:45+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["pt"]
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
## 简介
主要使用基于 Debian 的 VPS 进行发种，流程上与纯粹的 Windows 平台发种略有不同，需要依赖命令行工具进行处理。

## mediainfo

执行 `sudo apt install mediainfo` 即可安装 mediainfo，执行 `mediainfo video_name` 即可生成对应的 mediainfo。

可以利用 `>` 将输出重定向到指定文件。

## 视频截图

海豹要求三张原始分辨率的视频截图，我们可以利用 ffmpeg 来抓取截图。

执行 `sudo apt install ffmpeg` 即可安装 ffmpeg。

通过执行 `ffmpeg -ss 00:08:06 -i $video_file -vframes 1 -f image2 -y test1.png` 即可生成原始分辨率的 png 截图，其中 `$video_file` 要替换成对应的文件名。

## 生成种子

执行 `sudo apt install mktorrent` 安装 mktorrent，执行 `mktorrent`

## 脚本

将 mediainfo 和 视频截图整合成了一个简单的 bash 脚本：

```sh
#!/bin/bash

video_file=""

# 遍历当前目录下所有 .mp4 或者 .mkv 结尾的文件

for file in *.mp4 *.mkv; do
	if [ -f "$file" ]; then
		video_file+="$file"
	fi
done
echo "Video files: $video_files"

mediainfo "$video_file" >log.txt
#
ffmpeg -ss 00:08:06 -i $video_file -vframes 1 -f image2 -y test1.png
ffmpeg -ss 00:06:06 -i $video_file -vframes 1 -f image2 -y test2.png
ffmpeg -ss 00:10:06 -i $video_file -vframes 1 -f image2 -y test3.png
```

该脚本目前仅适用于目录下只有一个要做种的 mkv 文件的情况。
