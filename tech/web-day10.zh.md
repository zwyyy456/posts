---
title: "Web Day10：加入线程池到服务器"
date: 2023-11-18T15:19:11+08:00
lastmod: 2023-11-18T15:19:11+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cpp", "web server"]
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
## 前言

到 day9 的时候，一个单线程的服务器已经算写好了。Reactor 驱动大致成型。

服务器的启动流程大致如下，先创建 EventLoop 对象 `loop`（里面包含了 Epoll 对象），然后 Server 会利用 loop 实例化对象 `server`，Server 对象实例化时，Acceptor 类型的 `acceptor_` 对象会被初始化, Acceptor 对象用于建立连接，会在 Server 的构造函数里回调函数 `acceptor_->new_conn_callback_` 会被注册为 `server->NewConn(Socket *clnt_sock)` ，由 `acceptor_->AcceptConn()` 来调用。而 Acceptor 的构造函数中 Channel 类的实例 `accept_ch_` 会被初始化，而回调函数 `acceptr_ch->callback_` 会被注册为 `acceptor_->AcceptConn()`，最终还是调用的 `server->NewConn(Socket *clnt_sock)`。

```cpp
Server::Server(EventLoop *loop) :
    loop_(loop), acceptor_(nullptr) {
    acceptor_ = new Acceptor(loop);
    std::function<void(Socket *)> callback = [this](auto &&PH1) { NewConn(std::forward<decltype(PH1)>(PH1)); };
    acceptor_->set_new_conn_callback(callback);
}

void Server::NewConn(Socket *sock) {
    auto *conn = new Connection(loop_, sock); // 这里应该是 clnt_sock
    std::function<void(Socket *)> callback = [this](auto &&PH1) { DeleteConn(std::forward<decltype(PH1)>(PH1)); };
    conn->set_delete_conn_callback(callback);
    connections_[sock->getfd()] = conn;
}

void Acceptor::AcceptConn() {
    auto *clnt_addr = new InetAddress();
    auto *clnt_sock = new Socket(sock_->Accpet(clnt_addr));
    printf("new client fd %d! IP: %s Port: %d\n", clnt_sock->getfd(), inet_ntoa(clnt_addr->get_addr().sin_addr), ntohs(clnt_addr->get_addr().sin_port));
    clnt_sock->Setnonblocking();
    new_conn_callback_(clnt_sock);
    delete clnt_addr;
}

void Acceptor::set_new_conn_callback(std::function<void(Socket *)> &callback) {
    new_conn_callback_ = callback;
}
```

当建立连接的时候，会生成一个 Connection 实例，存储在 Sever 的 `connections_` 变量中。`connections_` 是 key 为 client fd，value 为建立的连接的指针 `Connection *`。并在，在建立连接时，`Server::DelteConn` 会被注册为 Connection 实例的回调函数 `delete_conn_call_back_`。当读取数据时，如果发现 `read_bytes == 0`，就会执行 `delete_conn_call_back_` 来断开连接。

```cpp
void Server::DeleteConn(Socket *sock) {
    Connection *conn = connections_[sock->getfd()];
    connections_.erase(sock->getfd());
    delete conn;
}
```

而 Connection 实例 `conn` 建立时，会创建一个 Channel 类的实例 `clnt_ch`， `clng_ch->callback_` 被设置为 `conn->echo()`。调用 `clnt_ch->EnableReading()` 会将该 `clnt_ch->fd` 添加到 epoll 关注的文件描述符中去，对应的 `ev` 中会包含指向该 `clnt_ch` 的指针。

之后便是 `loop->Loop()` 一直循环了。

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

最后注意一下生命周期的管理，`server` 的析构函数会 `delete acceptor_`，而 `acceptor_` 的析构函数 会 `delete sock_, addr_, accept_ch_`。`loop` 的析构函数会 `delete ep_`

断开连接时，会 `delete conn`，从而 `delete channel_, sock_, read_buffer_`。

在 main 函数的 `loop->Loop()` 循环结束时，会 `delete server` 和 `delete loop`。 

理论上 `loop->Loop()` 一直不会结束。

## 添加线程池

线程池说到底就是一个 `vector<std::thread>` 加上一个任务队列组成。Channel 可以说是生产者，每次发生事件，就往任务队列中添加要执行的函数 `callback_`，添加完之后要唤醒因条件变量阻塞的线程（`notify_one` 或者 `notify_all`）。而线程创建的时候，绑定的函数就是往任务队列里面取任务，如果队列为空，就会阻塞在条件变量 `cv_` 上，而释放锁。

```cpp
ThreadPool::ThreadPool(int size) :
    stop_(false) {
    for (int i = 0; i < size; ++i) {
        // 线程创建成功
        printf("create thread!\n");
        auto func = [this]() {
            while (true) {
                std::function<void()> task;
                {
                    std::unique_lock<std::mutex> lock(tasks_mtx_);
                    cv_.wait(lock, [this]() {
                        return stop_ || !tasks_.empty();
                    });
                    if (stop_ && tasks_.empty()) {
                        return;
                    }
                    task = tasks_.front();
                    printf("get task\n");
                    tasks_.pop();
                }
                printf("run task!\n");
                task();
            }
        }; // 创建线程时绑定的函数
        threads_.emplace_back(func);
    }
    printf("thread count: %d\n", threads_.size());
}

void Channel::HandleEvent() {
    loop_->add_thread(callback_);
    /* callback_(); */
}

void EventLoop::add_thread(std::function<void()> func) {
    thread_pool_->add_task(func);
}

void ThreadPool::add_task(std::function<void()> func) {
    {
        std::unique_lock<std::mutex> lock(tasks_mtx_);
        printf("add task\n");
        if (stop_) {
            throw std::runtime_error("ThreadPool already stop, can't add task any more");
        }
        tasks_.emplace(func);
        cv_.notify_one(); // 记得要唤醒线程
    }
}

```

说到底，就是把本来应该由 Channel 直接执行的任务，给挪到任务队列里面让线程池中的线程取出来去执行。

这里还有一个问题，那就是 Acceptor 的建立连接的任务，Channel 也会放到线程池里面让线程竞争得到执行权再执行，这样是不应该的。应该直接由主线程负责。





