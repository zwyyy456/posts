---
title: "N1 安装 Armbian 简单教程"
date: 2022-12-09T12:33:32+08:00
lastmod: 2022-12-09T12:33:32+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["tips", "geek", "armbian"]
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
## 前置

这一步非必须，如果之前 N1 已经刷了 OpenWrt 或者 Armbian 那么就不需要了，否则最好还是先刷入 [webpad 的官改 V2.2 固件](https://github.com/ophub/kernel/releases/download/tools/android_tv_phicomm_n1_mod_by_webpad_v2.2.tar.xz) 解压后利用双公头 usb 线和 usb-burning-tool 刷入到 N1 中。

具体步骤如下：

1. usb-burning-tool 导入 webpad 2.2 线刷包，勾选 `擦除 flash`，取消勾选 `擦除 bootloader`；
2. 点击 `开始` 按钮；
3. 3 秒内让 N1 通电，识别成功自动开始刷入；
4. 完成后拔电再上电，让 N1 重启；

当 N1 能正常开机之后，用鼠标开启 N1 的开发者模式（类似安卓手机，连续点击系统版本号即可）。

## 制作镜像
选择[Armbian_23.02.0_amlogic_s905d_bullseye_6.0.11_server_2022.12.08.img.gz](https://github.com/ophub/amlogic-s9xxx-armbian/releases/download/Armbian_bullseye_12.08.0628/Armbian_23.02.0_amlogic_s905d_bullseye_6.0.11_server_2022.12.08.img.gz)，下载好之后，解压，利用`rufus`刷入 u 盘。

## 安装
由于盒子在刷入`armbian`前为安卓系统，已开启`adb`，mac 的终端执行`adb connect 192.168.123.193`连接无线`adb`，`192.168.123.193`修改成 N1 的实际的 ip 地址，然后执行`adb reboot update`(这些过程最好都在 N1 外接显示器的环境下进行)，在显示器黑屏的瞬间将前一步的 u 盘插入到靠近 hdmi 的 usb 接口;

接下来显示器会跑一系列`starting xxx`的服务，直到最后让`login`的时候，应该是要先输入用户名`root`，再输入`1234`(默认密码)，然后输入两次重复的自定义密码 (如 x12x12);

然后会让你创建用户，可以按`Ctrl+C`跳过;

当显示已经启动完成，让你再登录之后，输入用户名`root`和前一步的自定义密码`x12x12`，即可进入命令行，然后执行`nand–sata-install`命令将系统刷写进 N1 的 emmc 中。

> 新的 armbian 镜像的刷入命令改成了 `armbian-install`，见 github release 的说明

## 安装好后的配置

### 添加用户

执行 `adduser zwyyy` 创建用户并执行`usermod -aG sudo username`添加到`sudo`用户组，之后的命令都在`zwyyy`用户下执行;

### 更换清华源
首先执行`sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak`进行备份，然后执行`echo > /etc/apt/sources.list`清空`sources.list`文件，然后执行`sudo vi /etc/apt/sources.list`，按下`i`进入`INSERT`模式，复制以下内容到`sources.list`中，然后执行`:wq`保存并退出;
```sh
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
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

然后在 root 用户下执行 

```sh
sed -i.bak 's#http://apt.armbian.com#https://mirrors.tuna.tsinghua.edu.cn/armbian#g' /etc/apt/sources.list.d/armbian.list
apt update
```

最后执行`sudo apt update`。

### 网络配置
由于本人在实验室，无法进入路由器后台查看，这里只能考虑设定静态 ip，然而设置好了静态 ip 之后，无法联网，只能先使用 dhcp，如果是家中，可直接进入路由器后台管理界面绑定和 mac 和 ip;

刷好了 armbian 的 N1 在每次重启之后，mac 会发生变化，因此首先执行`ip addr`，然后执行`ifconfig | grep ether`，其中一个是 wan 口的 mac，另一个是 lan 口的 mac，记下 lan 口的 mac，我这里是`9e:61:65:69:d7:aa`;

执行`sudo cp /etc/network/interfaces bak/network_interfaces.bak`备份文件，将文件内容修改为，address 和 hwaddres ether 根据机器自身的 mac 地址和 ip 地址来决定。
```
source /etc/network/interfaces.d/*

# Network is managed by Network manager
# You can choose one of the following two IP setting methods:
# Use # to disable another setting method


# 01. Enable dynamic DHCP to assign IP
#auto eth0
#iface eth0 inet dhcp
hwaddress ether 9E:61:A6:2B:7C:AA


# 02. Enable static IP settings(IP is modified according to the actual)
auto eth0
allow-hotplug eth0
iface eth0 inet static
address 192.168.6.103
netmask 255.255.255.0
gateway 192.168.6.1
dns-nameservers 192.168.6.1


# 03. Docker install OpenWrt and communicate with each other
#allow-hotplug eth0
#no-auto-down eth0
#auto eth0
#iface eth0 inet manual
#
#auto macvlan
#iface macvlan inet dhcp
#        hwaddress ether 9E:61:A6:2B:7C:AA
#        pre-up ip link add macvlan link eth0 type macvlan mode bridge
#        post-down ip link del macvlan link eth0 type macvlan mode bridge
#
#auto lo
#iface lo inet loopback
```

### 固定 ip 和 mac 地址
本机 mac: `9e:61:81:68:8f:aa`，每次重启之后 mac 会发生变化，因此考虑固定住 mac 地址;



## docker
### 安装 docker
参照[tuna docker 镜像源使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)

首先安装依赖：

```sh
sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

创建`/etc/apt/keyrings`文件夹，然后信任`Docker`的`GPG`公钥：

```sh
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

添加软件仓库：

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

执行安装：

```sh
sudo apt update
sudo apt install docker-ce
```

建立`docker`用户组：

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
```

设置`docker hub`中科大镜像源：

```sh
sudo vim /etc/docker/daemon.json
```

文件中加入：

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```

### 安装 portainer

```sh
docker volume create portainer_data
docker run -d -p 9000:9000 --name portainer -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/portainer_data portainer/portainer:linux-arm64
```

### 容器配置

docker 中配置 openwrt 作旁路由，由于实验室有线和 Wi-Fi 不在一个 ip 段，暂时放弃

armbian 配置 kvm 虚拟 openwrt

### kvm 安装 openwrt

暂时，也许以后也用不上，留个记录吧。

#### armbian 安装依赖包
参照 unifreq 大佬的教程[在 KVM 虚拟机中安装使用 OpenWrt 的说明](https://github.com/unifreq/openwrt_packit/blob/master/files/qemu-aarch64/qemu-aarch64-readme.md)，首先安装 KVM 依赖包，虽然 unifreq 提供的是基于`ubuntu jammy`的依赖列表，尽管我是基于`debian`的，但还是都装上这些依赖吧：
```
sudo apt-get install -y gconf2 qemu-system-arm qemu-utils qemu-efi ipxe-qemu libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager seabios vgabios gir1.2-spiceclientgtk-3.0 xauth at-spi2-core
```

确定 armbian 上 ssh 服务端开启了`X11Forwarding`功能：
```sh
# vi /etc/ssh/sshd_config 文件
X11Forwarding yes
# 如果之前未开启，保存配置文件后重启sshd
sudo systemctl restart sshd
```
这里 armbian 已经默认开启

将用户添加到 libvirt
```sh
# root 用户下
groupadd libvirt
usermod -aG libvirt zwyyy
```

#### 安装服务端和客户端
windows 安装 VcXsrv、putty，启动 Xlaunch 和 putty，putty 启动时，勾选 ssh-x11-enable x11 forwarding，ssh 连接到加入了 libvirt 用户组的用户：
```sh
# -X 选项开启X11 Forwarding
ssh -X zwyyy@host
# 运行远程GUI程序，界面将在windows电脑上显示出来
virt-manager
```

#### armbian 配置桥接网络
参照[debian10 使用 kvm 虚拟机并配置桥接网络](https://www.cnblogs.com/DouglasLuo/p/12731591.html)，另外可以参照[debian-kvm-wiki](https://wiki.debian.org/KVM)

用`brctl`命令创建桥接接口并管理桥接接口：
```sh
sudo brctl addbr br0 # 创建一个名为br0的桥接接口
sudo brctl show # 列出系统上所有桥接接口
```
将 armbian 的网卡接口加入到刚刚创建的`br0`桥接接口中：
```sh
sudo brctl addif br0 eth0
```
删除物理网卡接口的 ip 地址，把物理网卡接口的 ip 地址配置到桥接接口上，并开启桥接接口，然后添加默认网关：
```sh
sudo ip addr del dev eth0 10.80.17.82/24
sudo ip addr add 10.80.17.82/24 dev br0
sudo ip link set up br0
sudo route add default gw 10.80.17.1
```
如果要恢复原来的状态，只需要将桥接接口关闭，然后从桥接接口中删除物理网卡接口即可：
```sh
sudo ip link set br0 down
sudo brctl delif br0 eth0
sudo ip link set eth0 down
sudo ip link set up eth0 #重启物理网卡
```

#### 安装过程截图
上传`qemu`固件镜像`op.qcow2`(下载自 unifreq 的 tg 频道，解压后改名为`op.qcow2`):
```sh
scp Downloads/openwrt/op.qcow2  zwyyy@10.80.17.82:/home/zwyyy/op_kvm
```
安装过程截图参照 unifreq

安装好之后，列出虚拟机列表
```sh
sudo virsh list --all # 或者root用户执行，否则只有空
```
