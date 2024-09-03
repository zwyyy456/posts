---
title: "RHEL 7 个人开发环境部署"
date: 2024-08-21T23:22:48+08:00
lastmod: 2024-08-21T23:22:48+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["linux", "geek"]
description: "" #描述
weight: # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---

## 前言

入职了某公司，福利待遇不错，就是这开发环境属实一言难尽，开发机部署在内网，没有网络，只能通过内外网交换软件传文件，没有 root 权限（暂无，后面拿到 root 权限我也不敢用来安装什么软件），开发机的系统版本还特别旧，是 2014 年发布的 RHEL 7，上面的软件版本也是老掉牙了（除了没人用的 cmake），令人无语。故这里记录一下我克服困难，在这个 RHEL 7 上配置开发环境的步骤。

> 注，考虑到安全问题，以及权限受限，软件都安装在 `$HOME/.local` 目录下，然后设置对应的环境变量，其中 bash 不会去设置环境变量。

整体部署思路如下，利用自己的 VPS 开了一个 CentOS 7 的 docker 镜像，并创建一个与开发机上本人用户一致的用户，并使其 `$HOME` 目录与开发机的一致，在 CentOS 7 上，通过源码编译安装到 `$HOME/.local` 目录下，然后将 `$HOME/.local` 目录拷贝到开发机上，再配置好对应的环境变量，如果再不行，再考虑把源码以及依赖的源码拷贝到开发机上，再进行编译安装。

可以通过执行 `ldd ${binary_name}` 来查看运行该程序需要哪些动态链接库。 

## 软件安装

### GCC

执行以下命令即可安装 GCC：

```sh
mkdir src

# 安装依赖库 GMP
cd $HOME/src
wget https://gmplib.org/download/gmp/gmp-6.2.1.tar.xz
tar -xvf gmp-6.2.1.tar.xz
cd gmp-6.2.1

./configure --prefix=$HOME/.local
make -j$(nproc)
make install

# 安装依赖库 MPFR
cd $HOME/src
wget https://www.mpfr.org/mpfr-current/mpfr-4.1.0.tar.xz
tar -xvf mpfr-4.1.0.tar.xz
cd mpfr-4.1.0

./configure --prefix=$HOME/.local --with-gmp=$HOME/.local
make -j$(nproc)
make install

# 安装依赖库 MPC
cd $HOME/src
wget https://ftp.gnu.org/gnu/mpc/mpc-1.2.1.tar.gz
tar -xvf mpc-1.2.1.tar.gz
cd mpc-1.2.1

./configure --prefix=$HOME/.local --with-gmp=$HOME/.local --with-mpfr=$HOME/.local
make -j$(nproc)
make install

# 可能还需要安装 m4

# 编译安装 GCC
cd $HOME/src
wget https://ftp.gnu.org/gnu/gcc/gcc-11.2.0/gcc-11.2.0.tar.xz
tar -xvf gcc-11.2.0.tar.xz
cd gcc-11.2.0

# 下载 GCC 依赖，由于前面已经安装好了依赖，这一步应该是不需要了
./contrib/download_prerequisites

mkdir build
cd build

## 配置 GCC 编译
../configure --prefix=$HOME/.local/gcc-11.2 --disable-multilib --enable-languages=c,c++ --with-gmp=$HOME/.local --with-mpfr=$HOME/.local --with-mpc=$HOME/.local

## 编译并安装
make -j$(nproc)
make install

## 设置环境变量，.bashrc 不设置
echo 'export PATH=$HOME/.local/gcc-11.2/bin:$PATH' >> ~/.zshrc
echo 'export LD_LIBRARY_PATH=$HOME/.local/lib:$LD_LIBRARY_PATH' >> ~/.zshrc
echo 'export LIBRARY_PATH=$HOME/.local/lib:$LIBRARY_PATH' >> ~/.zshrc
source ~/.bashrc
```

### OpenSSL

安装 Python 3.10 需要高于 1.11.1 的 OpenSSL，机器上的 OpenSSL 版本太旧了，故重新编译安装。

```sh
## 安装 perl-devel
wget https://www.cpan.org/src/5.0/perl-5.34.0.tar.gz
tar -xzf perl-5.34.0.tar.gz
cd perl-5.34.0
./Configure -des -Dprefix=$HOME/.local/perl5
make
make install
export PATH=$HOME/.local/perl5/bin:$PATH
export PERL5LIB=$HOME/.local/perl5/lib/perl5:$HOME/.local/perl5/lib64/perl5:$PERL5LIB
export PERL5LIB=$HOME/.local/perl5/share/perl5:$PERL5LIB

cd $HOME/src
wget https://github.com/openssl/openssl/releases/download/openssl-3.3.1/openssl-3.3.1.tar.gz
tar -zxvf openssl-3.3.1.tar.gz
cd openssl-3.3.1
./config --prefix=$HOME/.local/openssl --openssldir=$HOME/.local/openssl shared zlib
make -j$(nproc)
make install

### CMake

执行以下命令以编译安装 CMake。

```sh
wget https://github.com/Kitware/CMake/releases/download/v3.30.2/cmake-3.30.2.tar.gz
mv cmake-3.30.2.tar.gz $HOME/src
cd $HOME/src
tar -xvf cmake-3.30.2.tar.gz
cd cmake-3.30.2
./bootstrap --prefix=$HOME/.local/cmake -- -DOPENSSL_ROOT_DIR=$HOME/.local/openssl -DOPENSSL_INCLUDE_DIR=$HOME/.local/openssl/include -DOPENSSL_LIBRARIES=$HOME/.local/openssl/lib
make -j2
make install
```

### Python

编译安装 Python 3.10 需要用到我们编译安装的 OpenSSL。

### llvm17

执行以下命令编译安装 llvm17，需要使用我们自行编译安装的 gcc。注意编译 llvm 需要 gcc7.4.0+ 和 python3.6+。

```sh
git clone -b llvmorg-17.0.6 --depth=1 https://github.com/llvm/llvm-project.git

## 设置环境变量
export CC=$HOME/.local/gcc-11.2/bin/gcc
export CXX=$HOME/.local/gcc-11.2/bin/g++
export LD_LIBRARY_PATH=$HOME/.local/gcc-11.2/lib64:$LD_LIBRARY_PATH
export PATH=$HOME/.local/gcc-11.2/bin:$PATH

mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$HOME/.local/llvm17 -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" -G "Unix Makefiles" ../llvm
```


