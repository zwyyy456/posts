---
title: "通过 frp 使用 ssh 连接内网服务器"
date: 2023-04-02T17:41:53+08:00
lastmod: 2023-04-02T17:41:53+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["geek", "ssh", "tips"]
description: "" #描述1
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
## 配置frp
### 安装frp
`~/Prog`目录下执行`wget https://github.com/fatedier/frp/releases/download/v0.48.0/frp_0.48.0_linux_amd64.tar.gz`下载压缩包，然后执行`tar -zxvf file.path`来解压，将解压生成的文件夹重命名为`frp`。

### 设置`frpc.toml`
修改`frpc.toml`文件为以下内容:
```
serverAddr = "nj1.mossfrp.cn"
serverPort = 51960
token = "3nj117667696278235715"
tls_enable = true
use_encryption = true

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22333
remotePort = 51962

[[vscode]]
name = "vsc"
type = "tcp"
localIP = "127.0.0.1"
localPort = 5433
remotePort = 51963

[[qbit]]
name = "qbit"
type = "tcp"
localIP = "127.0.0.1"
localPort = 28080
remotePort = 51968

[[qbit]]
name = "iyuu"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8787
remotePort = 51967

```

## 利用`systemctl`实现`frp`自启动
执行`sudo touch /etc/systemd/system/frp.service`，修改文件内容为
```
[Unit]
Description=frp client
After=network-online.target

[Service]
Type=notify
Restart=always
RestartSec=60
ExecStart=/home/zwyyy/Prog/frp/frpc -c /home/zwyyy/Prog/frp/frpc.ini # 注意这里不能使用~为路径

[Install]
WantedBy=multi-user.target
```
然后执行 `sudo systemctl enable frp.service` 和 `sudo systemctl start frp.service`。


**以下说明来自ChatGPT！**

“[Install]”部分提供了有关如何安装此系统服务的信息。

在Install部分中，“WantedBy”属性确定了哪个级别的目标(或多个级别的目标)应用于此服务。多个级别的目标可以在逗号分隔列表中指定。

通常，系统管理员会将服务添加到“multi-user.target”，这将确保在系统启动时服务自动启动，并在用户登录时保持运行。

“WantedBy=multi-user.target”表示此服务应该在系统启动时启动，并与多个用户有关的目标相关联，这样就可以在多个用户登录时持续运行服务。

此外，`[Install]`部分还提供了以下命令，可以用来启动和停止服务：

- `sudo systemctl start frp.service`：启动服务
- `sudo systemctl stop frp.service`：停止服务

