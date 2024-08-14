---
title: "Win11 重装记录"
date: 2024-03-03T11:45:50+08:00
lastmod: 2024-03-03T11:45:50+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["win11", "dev"]
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

## 起因

最近一两个月，时常碰到笔记本电脑死机的情况，具体表现为电脑卡死没反应，同时发出嗡嗡的电流声，前两天还出现了电脑蓝屏的情况，错误码为 "Clock Watchdog Timeout"，网上找了一圈，有不少人碰到了和我一样的问题，有个解决方案说要在电源模式下设置硬盘永不休眠，我尝试了，并没有用，推测可能的原因有以下几种：

1. 笔记本过热，一般都是一边看视频一边打游戏时出现卡死的情况，并且我的笔记本电脑是外接了 4K 显示器在用，卡死的时候，能感觉到笔记本非常烫；

2. 固态掉盘，用的海康威视 c2000 pro，据说有掉盘的先例；

3. 奇奇怪怪的驱动、硬件兼容性问题；

正好，这个笔记本电脑的系统已经很久没有重装过了，里面的东西、配置也被我搞得挺乱了，干脆重装一遍系统，看看是否还有问题。

这篇文章主要是记录一下重装系统的过程和一些重要步骤以及要安装的软件。

## 备份

需要备份的文件与文件夹都在 `C:\Users\zwyyy` 目录下，手动把要备份的文件夹打包，拷贝到了我的移动硬盘中去。

## 制作系统安装 U 盘

首先，在微软官网下载 64 位的 [Win11 系统镜像](https://www.microsoft.com/software-download/windows11)，下载好之后，利用 Rufus 软件将镜像写入到 U 盘中去。

然后是很重要的一步，如果直接开始安装系统，在安装过程中会发现无法识别 U 盘，对 11 代及以上的 Intel CPU 的笔记本，要识别 NVME 协议的固态硬盘，需要安装一个名为 Intel Rapid Storage Technology（IRST）的驱动程序，可以去笔记本厂商的官网下载对应的驱动，将驱动压缩包解压到系统安装 U 盘中即可，在安装系统，选择将 Windows 安装在哪里时，点击加载驱动程序，选择之前驱动程序解压到的那个目录，即可找到对应的驱动程序，加载了驱动程序之后，就能识别到硬盘了，之后的安装就是正常按照指引来就行了。

> acer 下载驱动需要先下载一个序列号检测识别软件，识别到序列号之后，就能下载电脑型号的对应驱动程序了。

> 我的笔记本型号为 SWIFT SF314-511

## 驱动安装

重装好系统之后发现触控板和指纹不可用，需要安装驱动，打开 `设置 -> Windows 更新 -> 高级选项 -> 可选更新 -> 驱动程序更新`，即可安装这些驱动程序。

## 软件安装

### Scoop

打开 Microsoft Store，安装 Windows Terminal，然后将 Shell 设置为 Windows PowerShell，执行以下两条指令：

```sh
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUse
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
``` 

就会自动帮你安装 Scoop，然后就能通过 Scoop 安装软件了，用起来类似 Mac 的 Homebrew。

### 启用小鹤双拼

手动添加注册表项目。

1. `Windows + R` 调出命令运行框，输入：`regedit` 后确认打开注册表；

2. 打开以下路径：`HKEY_CURRENT_USER\Software\Microsoft\InputMethod\Settings\CHS`

3. 新建字符串值，字符串名称为：`UserDefinedDoublePinyinScheme0`，值为：`小鹤双拼*2*^*iuvdjhcwfg^xmlnpbksqszxkrltvyovt`

4. 退出后进入微软拼音设置，即可看到小鹤双拼方案已存在，选择保存即可。

### 编程相关

首先，为 Scoop 添加 extras 库，执行 `scoop install git` 和 `scoop bucket add extras` 即可，如果要安装字体，执行 `scoop bucket add nerd-fonts`。

然后，执行以下命令安装软件：

```sh
scoop install vscode
scoop install sublime-text
scoop bucket add nerd-fonts
scoop install Cascadia-Code
scoop install miniconda
scoop install LXGWWenKai
scoop install neovim
scoop install wezterm
scoop install neovide # 利用 Rust 编写的 neovim 跨平台 GUI
scoop install llvm # C/C++ 的运行环境
scoop install pwsh # 新一代 powershell
scoop install FiraMono-NF-Mono
scoop install picgo # 截图并上传图床

scoop bucket add j178 https://github.com/j178/scoop-bucket.git
scoop install j178/leetgo
```

Neovim 的配置见 [Neovim 的配置与使用](https://blog.zwyyy456.tech/zh/posts/blog/neovim_tutorial/)。

Sublime Text4 的配置见 [配置 Sublime Text4为 C++ 编辑器](https://blog.zwyyy456.tech/zh/posts/blog/sublime_cpp/)。

Wezterm 的配置见 [Wezterm 的配置与使用](https://blog.zwyyy456.tech/zh/posts/blog/wezterm_tutorial/)

Vscode 的配置见 [Vsocde 使用 clangd](https://blog.zwyyy456.tech/zh/posts/blog/clangd_vscode/) 以及博客中的其他相关内容。如果是手动下载安装的 pwsh，那么 vscode 会自动检测到这个 Powershell，创建终端时默认的 shell 就是 Powershell，但由于我的 Powershell 是通过 scoop 安装的，因此 vscode 无法自动检测到，如果希望让 vscode 的默认 shell 为 Powershell，那么需要在配置文件中添加如下内容：

```json
"terminal.integrated.profiles.windows": {
    "pwsh": {
        "icon": "terminal-powershell",
        "path": [
            "C:\\Users\\zwyyy\\scoop\\shims\\pwsh.exe"
        ]
    }
},
"terminal.integrated.defaultProfile.windows": "pwsh"
```

此外，vscode 新增了一项名为粘滞滚动的特性，可以点击 `View -> Appearance -> Sticky Scroll` 关闭

Microsoft Store 中可以安装 Debian，安装好 Debian 之后要先在 Windows Terminal 中执行 `wsl --install` 来安装 WSL。

此外，还可以利用 scoop 来安装字体，例如 `FiraMono Nerd Font Mono`。


### 办公

安装 Office 365，搜索 Microsoft 365，然后即可在官网下载安装工具，打开该安装工具之后，等待安装完成即可。

Microsoft Store 中可以免费安装 Pixpin 和 OneCommander，前者可以理解为 Snipaste 的替代，后者则是自带文件管理器的上位替代。

Edge 浏览器注意关闭安全 DNS，如下图所示:

![](https://pic-upyun.zwyyy456.tech/picgo/20240417220840.png)

通过 scoop 安装了 picgo 之后，其又拍云的图床配置如下图：

![](https://pic-upyun.zwyyy456.tech/picgo/20240417221055.png)

其中，bucket 应该是云存储设置的服务名称，加速域名和操作员就是又拍云中设置的对应的加速域名和操作员，Picgo 本身并不支持截图，只能将本地图片或者剪切板中的图片上传到图床，可以将 Pixpin 与 Picgo 组合起来使用。

我将 Picgo 的快速上传的快捷键设置为了 `Ctrl + super + u`。


### 游戏

海豚加速器，可以免费加速炉石传说 90 天。

ak 加速器，每天 00:00 ~ 14:00 可以免费加速。

安装炉石传说时，语言不能选择中文，否则安装进度条会卡住，最后失败。

### 笔记

安装 RemNote。

### 其他

安装 IDM 时，去官网下载并安装，然后进入 163 邮箱，查找荔枝数码发送的授权邮件，里面有关键的授权码，利用这个授权码即可完成注册。

OneCommander: 非常好用的文件管理器，可以在 Microsoft Store 中获取并安装。

qq & wechat：去腾讯官网下载即可。

> 注意，qq 与微信不要把聊天记录的存储路径设在 OneDrive 会同步的目录，否则会导致 OneDrive 的图片中存在一大堆缩略图。

Mega 网盘：注意去下载测试版。

Keepass：注意数据库的存放位置就行。

安装 ContexMenuManager 以管理右键菜单。

### 配置

可以将 GitHub 中的 `dotfile` repo 给 clone 到 `C:\Users\zwyyy` 目录下作为 `.config` 目录，许多程序，例如 `wezterm` 的配置文件都是位于 `C:\Users\zwyyy\.config` 目录，当然，neovim 的配置文件位于 `C:\Users\zwyyy\AppData\local\nvim` 目录中。







