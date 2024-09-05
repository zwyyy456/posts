---
title: "Web Day6：EventLoop 类与事件驱动"
date: 2023-11-15T20:09:00+08:00
lastmod: 2023-11-15T20:09:00+08:00 #更新时间
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
## 事件驱动

原先的代码中，不管是接受客户端连接还是处理客户端事件，都是围绕epoll来编程，可以说epoll是整个程序的核心，服务器做的事情就是监听epoll上的事件，然后对不同事件类型进行不同的处理。这种以事件为核心的模式又叫事件驱动，事实上几乎所有的现代服务器都是事件驱动的。和传统的请求驱动模型有很大不同，事件的捕获、通信、处理和持久保留是解决方案的核心结构。

## 将服务器改造成 Reactor 模式

我们将服务器抽象成一个 `Server` 类，类中有一个 main-reactor，main-reactor 的核心是一个 `EventLoop`，即不断循环，一旦有事件发生，我们就会通过 `ep->Poll` 知晓，然后作出对应的处理。

## Channel 的修改

day5 中，每个 channel 里面都含有一个 `Epoll *` 指针，表示它在哪个 `epoll` 实例中被关注，这里我们把 `Epoll *` 替换成了 `EventLoop *`，表示该 channel 处于哪个事件循环中，事实上 `EventLoop` 的关键成员就是 `Epoll *`。

## 初始化流程

首先创建 `EventLoop` 对象 `loop`，它会创建一个 Epoll 实例，核心就是 `epfd`，然后利用 `loop` 初始化 `Server` 对象，在这个过程中，会完成服务器的 `serv_fd` 的创建以及 `bind`，`listen`，同时将对应着 `serv_fd` 以及新建连接事件的 `Channel` 创建出来，将 `Channel` 的处理事件的回调函数设置为 `NewConn`；调用 `serv_ch->EnableReading` 会将关注的事件类型设置为 `EPOLLIN | EPOLLET` 并调用 `loop->UpdateChannel(this)` 最终调用 `ep->UpdateChannel(ch)`，从而将 `serv_ch` 对应的文件描述符添加到 epoll 关注的文件描述符列表或者修改。

```cpp
Server::Server(EventLoop *loop) :
    loop_(loop) {
    Socket *serv_sock = new Socket();
    InetAddress *serv_addr = new InetAddress("127.0.0.1", 8888);
    serv_sock->Bind(serv_addr);
    serv_sock->Listen();
    serv_sock->Setnonblocking();
    Channel *serv_ch = new Channel(loop, serv_sock->getfd());
    std::function<void()> cb = [this, serv_sock] { NewConn(serv_sock); };
    serv_ch->set_callback(cb);
    serv_ch->EnableReading();
}

void Channel::EnableReading() {
    events = EPOLLIN | EPOLLET;
    loop_->UpdateChannel(this);
}

void EventLoop::UpdateChannel(Channel *ch) {
    ep_->UpdateChannel(ch);
}

void Epoll::UpdateChannel(Channel *ch) {
    int fd = ch->getfd();
    struct epoll_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.data.ptr = ch;
    ev.events = ch->get_events();
    if (!ch->get_in_epoll()) {
        // 添加到 epoll 中
        errif(epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1, "epoll add error!\n");
        ch->set_in_epoll();
    } else {
        errif(epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev) == -1, "epoll mod error!\n");
    }
}

void Server::NewConn(Socket *serv_sock) {
    InetAddress *clnt_addr = new InetAddress();
    Socket *clnt_sock = new Socket(serv_sock->Accpet(clnt_addr));
    printf("new client fd %d! IP: %s Port: %d\n", clnt_sock->getfd(), inet_ntoa(clnt_addr->addr.sin_addr), ntohs(clnt_addr->addr.sin_port));
    clnt_sock->Setnonblocking();
    int sockfd = clnt_sock->getfd();
    Channel *clnt_ch = new Channel(loop_, sockfd);
    std::function<void()> callback = [this, sockfd] { HandleReadEvent(sockfd); }; // 设定建立了连接的 ch 对应的回调函数
    clnt_ch->set_callback(callback);
    clnt_ch->EnableReading();
}

```

上述代码就是有新连接到来时的函数调用流程。

## 新连接到来后

当有新连接到来后，就会调用 `ch->HandleEvent()`，即我们绑定的回调函数 `Server::NewConn(Socket *serv_sock)`，它会建立连接，利用 `accept` 返回的文件描述符，创建 `clnt_ch`，并设置 `clnt_ch` 的回调函数为 `Server::HandleReadEvent`。

这样，当活跃的 channel 是 `clnt_ch` 时，就会执行读写事件。

```cpp
void EventLoop::Loop() {
    while (!quit_) {
        auto chs = ep_->Poll(); // Poll 返回的是活跃的 Channel 的集合
        for (auto *ch : chs) {
            ch->HandleEvent();
        }
    }
}
```





