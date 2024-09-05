---
title: "Web Day8：抽象出 Connection 类"
date: 2023-11-16T15:25:27+08:00
lastmod: 2023-11-16T15:25:27+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["cpp", "web server"]
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
## Acceptor 类

一言以蔽之，Acceptor 类负责接受连接，调用 `AcceptConn`，到这里，接受连接已经完全由 `Acceptor` 类来负责。`Acceptconn` 会调用被注册的回调函数 `new_conn_callback_(clnt_sock)`，实际上就是调用 `Server::NewConn(clnt_sock)`，该函数会将 `{clnt_fd, conn}` 的键值对添加到 Server 类的 map 中去。

此外，还会为 Connection 类注册删除 Connection 的回调函数；

```cpp
void Acceptor::AcceptConn() {
    auto *clnt_addr = new InetAddress();
    auto *clnt_sock = new Socket(sock_->Accpet(clnt_addr));
    printf("new client fd %d! IP: %s Port: %d\n", clnt_sock->getfd(), inet_ntoa(clnt_addr->get_addr().sin_addr), ntohs(clnt_addr->get_addr().sin_port));
    clnt_sock->Setnonblocking();
    new_conn_callback_(clnt_sock);
    delete clnt_addr;
}

void Server::NewConn(Socket *sock) {
    auto *conn = new Connection(loop_, sock); // 这里应该是 clnt_sock
    std::function<void(Socket *)> callback = [this](auto &&PH1) { DeleteConn(std::forward<decltype(PH1)>(PH1)); };
    conn->set_delete_conn_callback(callback);
    connections_[sock->getfd()] = conn;

```

Acceptor 的回调函数的注册过程发生在 Server 类的创建过程中。

## Connection 类

完成连接之后，客户端文件描述符就绪之后的处理函数，定义在了 Connection 类中，一个 Connection 对应着一个 Channel，每有一个 Connection 类被创建出来，这个 Connection 对应的 Channel 的回调函数就会被注册为 Connection 类中的处理函数；

```cpp
Connection::Connection(EventLoop *loop, Socket *sock) :
    loop_(loop), sock_(sock), channel_(nullptr) {
    channel_ = new Channel(loop, sock->getfd());
    int fd = sock_->getfd();
    std::function<void()> callback = [this, fd]() {
        echo(fd); // Connection 类的事件处理函数
    };
    channel_->set_callback(callback); // 注册回调函数
    channel_->EnableReading();
}

void Channel::HandleEvent() {
    callback_();
}
```


