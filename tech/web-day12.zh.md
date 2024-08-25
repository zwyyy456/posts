---
title: "Web Day12：实现主从 Reactor 多线程模式"
date: 2023-11-19T13:00:39+08:00
lastmod: 2023-11-19T13:00:39+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cpp", "web server"]
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
## 前言

在 Day11 中，我们实现了一种最容易想到的 Reactor 多线程模式，即将每个 Channel 的任务分配给一个线程执行。

> 这个模式逻辑上有不少问题，例如线程池由 EventLoop 来持有，按理来说应该由 Server 类来管理，这是受到了 Channel 类的限制，Channel 类仅有 EventLoop 成员。

## 主从 Reactor 模式

主从 Reactor 模式有以下几个特点：

1. 服务器一般只有一个 main reactor，可以有很多个 sub reactor；
2. 服务器管理一个线程池，每个线程对应一个 sub reactor，每个 sub reactor 负责一部分 Connections 的事件循环，事件执行也在这个线程完成；
3. main reactor 只负责 Acceptor 建立新连接，然后将这个连接分配给一个 sub Reactor。

## Server 成员

/todo，测试 accept 和 connect 的时候区分非阻塞与阻塞

Server 的成员包括一个 main_reactor 和 多个 sub_reactors，每个 sub_reactor 对应一个独有的 EventLoop，每个 sub_reactor 由一个线程负责。这就是所谓的 **One Loop per Thread**。

```cpp
class Server {
  private:
    EventLoop *main_reactor_;
    Acceptor *acceptor_;
    std::map<int, Connection *> connections_;
    std::vector<EventLoop *> sub_reactors_;
    ThreadPool *thpool_;

  public:
    Server(EventLoop *evl);
    ~Server();

    void HandleReadEvent(int fd);
    void NewConn(Socket *serv_sock);
    void DeleteConn(int sockfd);
};
```

## main reactor 的工作流程

Server 创建的时候，会利用 main 函数的 `loop` 来初始化 Server，并利用 `loop` 来初始化 `main_reactor_`，和 `acceptor_`。 `acceptor_` 会有绑定了服务器 ip 和端口的 socket。初始化 `acceptor_` 的时候，会将 `acceptor_->new_conn_callback_` 注册为 `Server::NewConn(Socket *clnt_sock)`，当有连接时，`acceptor_` 调用 `Acceptor::AcceptConn()`，该函数会调用 `Socket::Accept(InetAddress *)` 来接受连接，以及调用 `new_conn_callback_(clnt_sock)`，实际上就是调用 `Server::NewConn(Socket *clnt_sock)`。

> `main_reactor_` 就是接受、建立连接的事件循环，

```cpp
Server::Server(EventLoop *loop) :
    main_reactor_(loop), acceptor_(nullptr) {
    acceptor_ = new Acceptor(main_reactor_);
    std::function<void(Socket *)> callback = [this](auto &&PH1) { NewConn(std::forward<decltype(PH1)>(PH1)); };
    acceptor_->set_new_conn_callback(callback); // 注册回调函数
    auto size = std::thread::hardware_concurrency(); // 获取 CPU 核心数?
    thpool_ = new ThreadPool(size);
    for (int i = 0; i < size; ++i) {
        sub_reactors_.push_back(new EventLoop());
    }

    for (int i = 0; i < size; ++i) {
        auto sub_loop = [capture0 = sub_reactors_[i]] { capture0->Loop(); };
        thpool_->add_task(sub_loop);
    }
}
Acceptor::Acceptor(EventLoop *loop) :
    loop_(loop), sock_(nullptr), accept_ch_(nullptr) {
    sock_ = new Socket();
    auto *addr = new InetAddress("127.0.0.1", 6789); // 6789679
    sock_->Bind(addr);
    sock_->Listen();
    sock_->Setnonblocking(); // Acceptor 建议使用阻塞式
    accept_ch_ = new Channel(loop_, sock_->getfd());
    std::function<void()> callback = [this] {
        AcceptConn();
    };
    // accept_ch_->set_use_threadpool(false); // 主 Acceptor 不使用线程池
    accept_ch_->set_read_callback(callback);
    accept_ch_->EnableReading();
    delete addr;
}
void Acceptor::AcceptConn() {
    auto *clnt_addr = new InetAddress();
    auto *clnt_sock = new Socket(sock_->Accpet(clnt_addr));
    printf("new client fd %d! IP: %s Port: %d\n", clnt_sock->getfd(), inet_ntoa(clnt_addr->get_addr().sin_addr), ntohs(clnt_addr->get_addr().sin_port));
    clnt_sock->Setnonblocking();
    new_conn_callback_(clnt_sock);
    delete clnt_addr;
}
void Server::NewConn(Socket *sock) {
    if (sock->getfd() != -1) {
        auto idx = sock->getfd() % sub_reactors_.size(); // 将连接随机分配到 sub_reactor
        auto *conn = new Connection(sub_reactors_[idx], sock); // 新建对应的 Connection 类
        auto callback = [this](auto &&ph1) { DeleteConn(std::forward<decltype(ph1)>(ph1)); };
        conn->set_delete_conn_callback(callback);
        connections_[sock->getfd()] = conn;
    }
}
```

一个连接对应一个 Connection 类，而 Connection 类创建的时候，会创建对应的 Channel，并注册 Channel 的回调函数，这个回调函数就是 Channel 用于处理读写事件的。

## sub_reactor 的工作流程

当 Server 被初始化时，会创建 k 个 sub_reactors，每个 sub_reactor 就是一个 EventLoop，这里没有什么 sub_acceptor。同时有 k 个子线程被创建出来，每个线程对应一个 sub_reactor。

子线程的任务很简单，每一个子线程对应着一个 EventLoop，任务就是一直执行 `sub_reactors[i]->Loop()`。

```cpp
void Server::NewConn(Socket *sock) {
    if (sock->getfd() != -1) {
        auto idx = sock->getfd() % sub_reactors_.size(); // 将连接随机分配到 sub_reactor
        auto *conn = new Connection(sub_reactors_[idx], sock); // 新建对应的 Connection 类
        auto callback = [this](auto &&ph1) { DeleteConn(std::forward<decltype(ph1)>(ph1)); };
        conn->set_delete_conn_callback(callback);
        connections_[sock->getfd()] = conn;
    }
}
```

> 始终要注意，一个 reactor 就是一个 EventLoop!

这里可以再注意一下建立连接的时候发生的事情，建立的 Conn 会根据取模的结果，分配给一个 sub_reactor，每个 sub_reactor 只会关系属于自己的那些 Connection。在 `Loop()` 中得到关注的 Connections 中的活跃的 Channel，然后执行这些 Channel 的回调函数。

> Channel 的回调函数在 Connection 创建的时候被注册。


