---
title: "Debian 开发环境配置"
date: 2023-11-17T16:16:27+08:00
lastmod: 2023-11-17T16:16:27+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["Debian", "dev"]
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

开发 web server 等项目，由于会使用到 Linux 的一些 API，用 Mac 来开发并不方便。我的 M1 Macbook Pro，由于采用了 arm 架构，使用虚拟机比较麻烦，并且虚拟机总觉得太笨重，不算方便。此外，也是由于 arm 架构的原因，使用 Docker 不方便，Docker Desktop 据说又卡又难用。

希望能有一个方案能同时解决以上两个问题（毕竟能用 docker 就能直接在 docker 里面跑 Debian 镜像了）。

经过一番搜索，发现了一个名为 [OrbStack](https://orbstack.dev/) 的工具，可以理解为 Mac 上的 WSL，至于实现原理，这里不去深究。

OrbStack 使用起来非常方便，`brew install orbstack` 之后，执行 `orbstack create debian orb-deb` 就可以创建名为 `orb-deb` 的虚拟机了，虚拟机和 macOS 之间的文件访问非常方便。Linux 中可以直接访问 macOS 中的文件和目录，macOS 中也一样。并且 CPU / 磁盘 / 内存都是按需使用的。

此外，可以在 Linux 虚拟机中非常方便地执行 macOS 的命令，在 macOS 中执行虚拟机 Linux 中的命令也同样如此。

后面的内容都是 Debian 开发环境的配置了，不论是 OrbStack 的 Debian 还是 WSL 又或者是物理机、VPS 都适用。

本博客内容会随时间而更新，也是我对 Debian 系统做的修改和配置的一个记录。

## 换源

阿里云的服务器可以无需执行此操作。

执行 `vi /etc/apt/sources.list`，将文件内容替换为以下内容

```sh
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware

deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
# deb-src https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

然后执行 `sudo apt update`。

## 创建 ssh 密钥并上传到 GitHub

执行 `ssh-keygen -t ed25519 -C "zwyyy456@hotmail.com"`，然后一路按下 `Enter`，密钥对存放在`~/.ssh`文件夹下，`id_ed25519.pub`为公钥，`id_ed25519`为私钥。

注意看提示，一路按 `Enter` 则 `passphrase` 为空。

然后在 GitHub 设置的 `SSH and GPG keys` 的那一栏，新建 `SSH Keys`，内容就填 `id_ed25519.pub` 里的内容。

## 安装软件

### 安装 fish

本人习惯于使用 fish 作为默认 shell。

执行 `sudo apt install fish && sudo chsh -s /usr/bin/fish` 即可安装并将默认 shell 设置为 fish。

执行 `git clone git@github.com:zwyyy456/dotfile.git ~/.config` 即可，该 repo 里面是我自己的一些配置文件。fish 的配置文件位于 `~/.config/fish/config.fish`。

### 配置 C++ 开发环境
参照 [VsCode 使用 clangd](https://blog.zwyyy456.tech/zh/posts/blog/clangd_vscode/)

执行 `sudo apt install clang clangd lldb cmake` 安装 C++ 开发环境。

### Lazygit

可以在 Lazygit 的 [GitHub Release](https://github.com/jesseduffield/lazygit/releases/) 界面下载对应的二进制包，以 `lazygit_0.40.2_Linux_arm64.tar.gz` 为例，利用 `wget` 下载后，执行 `tar -zxvf <filename>` 即可解压到当前目录。将可执行文件 `lazygit` 移动到 `/usr/local/bin` 即可。

可以为 `lazygit` 添加别名为 `lg`。对于 fish，只需要在 `~/.config/fish/config.fish` 中添加 `alias lg 'lazygit'` 即可。


### 安装 docker

由于 macOS 中的 Debian 本来就是虚拟出来的，要使用 docker 可以直接由 OrbStack 管理，因此这里只讨论 WSL 和物理机中的 Debian 安装 docker。

对 WSL，编辑 `/etc/wsl.conf` ，追加以下内容

```ini
[boot]
systemd=true
```

即可让 WSL 启用 systemd。

然后便能正常安装使用 docker 了。

参照[tuna docker镜像源使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)

首先安装依赖:

```sh
sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

创建`/etc/apt/keyrings`文件夹，然后信任`Docker`的`GPG`公钥:

```sh
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

添加软件仓库:
```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

执行安装:

```sh
sudo apt update
sudo apt install docker-ce
```

建立`docker`用户组:

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
```

设置`docker hub`中科大镜像源：

```sh
sudo vim /etc/docker/daemon.json
```

文件中加入:

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```


### Neovim

这里需要编译 Neovim，因为是 arm 架构，x86_64 架构建议直接下载 `.appimage` 然后移动为 `/usr/local/bin/nvim`。

执行以下命令：

```sh
sudo apt-get install ninja-build gettext cmake unzip curl
git clone https://github.com/neovim/neovim
cd neovim && git checkout stable
make CMAKE_BUILD_TYPE=RelWithDebInfo # 开始编译
cd build && cpack -G DEB # 生成 Debian 系的软件包
sudo dpkg -i xxx.deb # 看你生成的 .deb 包的包名
```

如果是 `.appiamge` 包，那么移动为 `/usr/local/bin/nvim` 之后还要执行 `sudo chmod +x /usr/local/bin/nvim` 和 `sudo apt install fuse3 libfuse2`。后者是安装 `.appimage` 运行所需的依赖。

然后执行 `git clone git@github.com:zwyyy456/nvim_config.git ~/.config/nvim`。

之后运行 `nvim`，LazyVim 就会安装好对应插件。

> 由于阿里的服务器连接 Github 效果也不好，因此这里选择为 GitHub 设置使用镜像：`git config --global url."https://mirror.ghproxy.com/https://github.com/".insteadOf "https://github.com/"`。

> 同时阿里的服务器 nvim 配置采用 onecloud 分支而非 main 分支；

至于 Neovim 的配置可以参考 [Neovim 的配置与使用](https://blog.zwyyy456.tech/zh/posts/blog/neovim_tutorial/)

### tmux

执行 `sudo apt install tmux` 安装 tmux，编辑 `~/.tmux.conf` 文件，追加以下内容：

```sh
# 设置默认shell为Fish
set -g default-shell /usr/bin/fish
set -g allow-passthrough on
```


## 博客的同步方案

整体来说通过 Filen 进行同步，免费用户通过邀请他人，最多可以获得 50GB 的存储空间，上传下载不限速不限流量。只是同步博客和代码文件已经够用了。

关键的一点是，Filen 同步文件是默认选项会排除 `.git` 这种以 `.` 开头的文件和文件夹，并且支持自定义 ignore（语法类似 `.gitignore`）。

Filen 还支持同步后再手动通过勾选的方式设置哪些文件夹之后不再同步。

> OneDrive 和 iCloud 至今都不能支持这一点。iCloud 也就罢了，毕竟只是用来备份照片的。OneDrive 从功能定位上来说应该也是同步盘，却连这种基础功能都无法支持，真是拉跨。


~~因此，这里我设置了 Filen 同步 `\\wsl.localhost\Debian\home\zwyyy\Documents\blog` 中的文件到云盘的 `blog` 目录，而 mac 则会同步 `~/Documents/blog` 目录到云盘的 `blog` 目录。~~

~~这里最好是设置仅同步 `content` 目录以及 `config.toml` 等关键配置文件。~~

```txt
*/archetypes/
*/assets/
*/i18n/
*/layouts/
*/static/
*/themes/
*/resources/
```

~~此外，博客会尽量及时 push 到 GitHub。~~

~~比较遗憾的是，Filen 不支持本地分别设置两个文件夹同步到同一个云盘目录。~~

Filen 在同步 WSL 中的文件时会出现问题，如下图所示：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065633.png)

因此暂时将博客存放于 `C:\Users\zwyyy\OneDrive\Documents\blog` 目录下，让 Filen 同步此目录，再通过 Git 同步到 WSL 中来，由于 OneDrive 同步无法忽略 `.git` 文件，因此尽量避免直接修改 `C:\Users\zwyyy\OneDrive\Documents\blog` 中的内容并提交到 Github。

对 mac 而言，博客直接存放在 `~/Documents/blog` 目录下并由 Filen 同步，iCloud 负责备份即可。

## Code 的同步方案

Code 的同步方案同上，对 mac 而言，放在 `~/Documents/code` 目录，iCloud 负责备份，Filen 负责同步，可以由 VsCode Remote 连接到 OrbStack 的虚拟机，并直接访问 mac 宿主机中的目录进行编辑。

此外，在 OrbStack 的虚拟机的目录中的代码文件，也可以被 Filen 顺利同步，这一点比起 WSL 来说要舒服很多。

## 安装 miniconda

