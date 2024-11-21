
# Linux

## docker 安装 centos7 镜像

执行以下命令

```sh
docker pull centos:7
docker tag centos:7 ct7
docker run -it ct7 /bin/bash
```

之后，便会进入到 centos7 容器内的 bash shell，由于 centos7 已经不再提供更新，连软件源都已经关闭，因此这里需要先换源，执行以下命令：

```sh
mv /etc/yum.repos.d/CentOS-Base.repo Centos-Base.repo.bak
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache && yum update -y
yum groupinstall -y "Development Tools"
yum install -y ncurses-devel gcc make wget tar
```

然后创建用户，并指定其 home 目录为 `/cffex/zengwy`​，命令如下：

```sh
mkidr /cffex && useradd -m -d /cffex/zengwy zengwy
yum install sudo && usermod -aG zengwy
```

接着执行以下命令下载 zsh 5.9 的源码并编译安装：

```bash
wget https://sourceforge.net/projects/zsh/files/zsh/5.9/zsh-5.9.tar.xz
tar -xvf zsh-5.9.tar.xz && cd zsh-5.9
./configure --prefix=$HOME/.local
make -j$(nproc) && make install
source ~/.bash_profile && zsh
wget https://github.com/starship/starship/releases/download/v1.21.1/starship-x86_64-unknown-linux-musl.tar.gz
tar -C ~/.local/bin -xvf starship-x86_64-unknown-linux-musl.tar.gz
mkdir .config
```

编辑 `~/.zshrc`​，修改其内容为常用配置内容，并在宿主机，通过 `docker cp ~/.config/starship.toml ${container_id}:/cffex/zengwy/.config`​ 将 starship 的配置文件移动到容器内，然后将 zengwy 用户的默认 shell 修改为 zsh

‍
