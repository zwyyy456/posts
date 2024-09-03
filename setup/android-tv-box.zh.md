---
title: "安卓电视盒子的折腾之旅"
date: 2024-06-11T22:39:23+08:00
lastmod: 2024-06-11T22:39:23+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["atv", "fun"]
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

## 前言

论文写完之后，以 840 元的价格购入了投影群里群友的优派 PJD7822HDL，只能说还能凑合用吧，噪音与发热巨大，开节能模式之后勉强可以接受，标称 3200 流明，实际上亮度不超过 50% 的情况下才能有比较好的对比度，亮度拉高了之后泛白严重，白天关灯拉窗帘看，由于我房间的窗帘的遮光性一般，感觉亮度还是低了点，晚上看倒不错，自带的音响效果也很一般（从闲鱼上淘了对音箱解决了）。不过好歹也是 dmd0.65，原生的 1080P 分辨率，清晰度还是让人满意的。

不过，该投影仪并无安卓系统，为了解决影视内容问题，我就开始在闲鱼上淘电视盒子，一共考虑了以下几个设备：移动魔百盒、亚马逊火棒、以及外贸盒子。

## 移动魔百盒

我是在 v2ex 上看到有人 60 出 cm211-1 增强版（2+16g），于是买下了它，事实证明这玩意现在最多值 50（因为我在闲鱼上 50r 又买了一个）。

这个魔百盒的 cpu 为晶晨 S905L3，默认固件是基于老掉牙的安卓 4.4，好在可以刷的固件还是不少的，我选择的是基于 Android TV 9.0 的 slimbox 固件，还是折腾了好一番才刷入的固件。

步骤如下，首先去淘宝购买了晶晨 S905 的免拆短接 HDMI 刷机工具，将它插在盒子的 HDMI 口，打开商家提供的刷机软件 USB Burning Tool ，和以前 N1 刷机的时候用的其实是同一个软件，该软件非绿色版，安装的时候会为你安装晶晨的相关驱动，安装好之后打开该软件，通过 USB 双头线连接电脑和魔百盒，打开 `USB_Burning_Tool`，会检测到已连接到设备，点击 `文件` -> `导入烧录包` 来导入 `img` 格式的固件，烧录配置选择默认的 `擦除 flash` 和 `擦除 bootloader` 即可，就可以开始烧录了，耐心等待即可。

> 由于该盒子已经刷过机了，按理来说不需要短接就能直接刷，也不需要什么免拆工具，强烈怀疑之前一直检测不到设备是因为用的绿色版，没有安装驱动。

刷好机之后，可以通过 Google play store 安装 Plex，理论上就能愉快玩耍了，然而我发现一个严重的问题，那就是用 Plex 播放音轨为 ac3 或者 eac3 格式的视频，会没有声音，经查找资料，应该是因为 S905L3 这个芯片不支持解码杜比全景声，而 Plex 默认对于这两种格式的音轨是在本地客户端直接硬解的，所以播放这样的视频会没有声音，而 Plex 的 tv 端界面无法像手机端那样针对音频格式设置是否硬解，因此，只能放弃这一款魔百盒，安装好 Tvbox 与 Emby，准备拿回去给家里人用了。

## 亚马逊火棒

经过查阅资料，发现亚马逊的 Fire tv stick 4k max（后面简称火棒），其对视频与音频格式的支持十分丰富，价格也还算能接受吧，一代 200 左右，二代 400 左右，其对比如下图所示：

![](https://pic-upyun.zwyyy456.tech/picgo/20240611234531.png)

可以看到二代相比一代的主要升级在于存储空间增加了 8g 和支持了更多音频格式，至于 cpu 性能提升个人认为不关键，毕竟视频解码能力没区别，也不会用盒子打游戏。

二代是支持 Dolby TrueHD 和 DTS-HD MA 的音频直通的，一代似乎不支持，这样看来，假如后面也搭建家庭影院，在不考虑观看双层杜比的原盘的情况下，二代火棒作为电视盒子也完全够用了。

然而，二代的火棒还是太贵了，一代又感觉比较老了，考虑到自己目前只是接这个垃圾投影仪看 Plex，又没必要上二代，因此将目光转向了外贸盒子。

## x98h pro

本来是准备在酷安上一个专门卖外贸盒子的贩子那里买的，后面在闲鱼上看到一个卖 x98H pro 的帖子，这个盒子被原主升级 magisk 之后卡开机界面了，于是我 80r 入手了该盒子。

该盒子芯片为全志的 H618，支持解码杜比全景声，支持解码 4K H265 和 VP9，可惜不支持硬解 av1，支持 HDR10+（杜比视界就别想了）。

刷机过程也经历了一番折腾，参照 [x98h pro盒子固件](https://www.znds.com/tv-1245405-1-1.html) 中的教程，教程作者也提供了刷机工具和固件，我刷的是 atv 固件，这里要注意的是，先安装驱动，如下图所示：

![](https://pic-upyun.zwyyy456.tech/picgo/20240612000506.png)

![](https://pic-upyun.zwyyy456.tech/picgo/20240612000723.png)

驱动位于 `x98hpro/PhoenixSuit/Drivers/AW_Driver`。

安装好驱动之后，先打开刷机软件，选择 `一键刷机`，并导入 img 格式的固件，此时左下角会显示“无设备连接”，然后给盒子通电，用牙签抵住 AV 接口孔中的复位键不放，最后用双公头 USB 线连接盒子和电脑（USB 线要插入**盒子后面的 USB 接口**），此时工具会显示设备已连接，点击 `立即升级` 即可开始刷机，之后可以松开牙签了。

![](https://pic-upyun.zwyyy456.tech/picgo/20240612000927.png)

教程作者提供的 ATV 固件源于俄罗斯的 slimbox 项目，最新固件可从 [slimbox x98h-pro](https://slimboxtv.ru/x98h-pro/) 处下载，不过下载要用 mega 盘，我嫌麻烦就用的之前教程作者的，后面也许可以再刷刷玩玩。

由于也是 slimbox，自带 Google play store，可以通过它安装 Plex，安装好之后，Plex 的设置中要将 `音频直通` 设置为禁用，否则播放音轨为 ac3 或者 eac3 的视频会卡住。

## 参考

[x98h pro盒子固件](https://www.znds.com/tv-1245405-1-1.html) 

[slimboxtv: x98h-pro](https://slimboxtv.ru/x98h-pro/)