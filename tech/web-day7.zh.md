---
title: "Web Day7：为服务器添加一个 Acceptor"
date: 2023-11-16T10:55:51+08:00
lastmod: 2023-11-16T10:55:51+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: [""]
tags: [""]
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
## Acceptor 定义

这里我的理解就是，一个 Acceptor 对应一个 EventLoop，当然也有一个独有 Channel，负责分发到对应的 epoll。

这里实际上是从 Server 中又分离出了一个 Acceptor，负责管理事件循环，服务器 socket 的创建与绑定，listen，同时由 Acceptor 来负责建立连接（逻辑上），调用的底层的建立连接的函数还是 `Server::NewConn(Socket *)`。

即实际上，Acceptor 类接管了之前 Server 类的工作。

```cpp
Acceptor::Acceptor(EventLoop *loop) :
    loop_(loop), sock_(nullptr), accept_ch_(nullptr), addr_(nullptr) {
    sock_ = new Socket();
    addr_ = new InetAddress("127.0.0.1", 8888);
    sock_->Bind(addr_);
    sock_->Listen();
    sock_->Setnonblocking();
    accept_ch_ = new Channel(loop_, sock_->getfd());
    std::function<void()> callback = [this] {
        AcceptConn();
    };
    accept_ch_->set_callback(callback);
    accept_ch_->EnableReading();
}


void Acceptor::AcceptConn() {
    new_conn_callback_(sock_);
}

void Acceptor::set_new_conn_callback(std::function<void(Socket *)> &callback) {
    new_conn_callback_ = callback;
}

Server::Server(EventLoop *loop) :
    loop_(loop), acceptor_(nullptr) {
    acceptor_ = new Acceptor(loop);
    std::function<void(Socket *)> callback = [this](auto &&PH1) { NewConn(std::forward<decltype(PH1)>(PH1)); };
    acceptor_->set_new_conn_callback(callback);
}

```

其中以下两行代码就是让 Acceptor 类可以调用 Server 类中的 `NewConn` 的关键：

```cpp
    std::function<void(Socket *)> callback = [this](auto &&PH1) { NewConn(std::forward<decltype(PH1)>(PH1)); };
    acceptor_->set_new_conn_callback(callback);
```

重申一遍，逻辑上由 Acceptor 类负责接受连接，但是底层还是由 Server 类来建立连接。
