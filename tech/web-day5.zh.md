---
title: "Web Day 5 添加 Channel 类"
date: 2023-11-15T15:29:00+08:00
lastmod: 2023-11-15T15:29:00+08:00 #更新时间
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

## Channel 类

`Channel` 类相当于将 `ep->addFd` 这一步拆成了两步，第一步是 `ch->enablereading`，它会调用 `ep->UpdateChannel(this)`，`this` 就是调用 `enablereading` 的那个 `ch`。

如何理解 Channel 类？可以认为每一个 ch 的实例，都对应着一个关注的文件描述符 `fd` 和一个要关注的事件类型 `events`，当前其实只有两类 Channel，一个是对应的服务器的 `fd`，另一类对应的是 `accept` 客户端的连接之后得到的 `fd`，`ch->events` 是 `fd` 所在的 Channel 实例需要关注的事件类型。

而 `active_events` 表示该 Channel 当前发生的事件类型，在 `ep->Poll()` 中会被设置。

```cpp
auto Epoll::Poll(int timeout) -> std::vector<Channel *> {
    std::vector<Channel *> active_channels;
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);
    errif(nfds == -1, "epoll wait error\n");
    active_channels.reserve(nfds);
    for (int i = 0; i < nfds; ++i) {
        Channel *ch = (Channel *)events[i].data.ptr;
        ch->set_active_events(events[i].events);
        active_channels.push_back(ch);
    }
    return active_channels;
}
```

而在 `ep->UpdateChannel(this)` 这个过程中，会创建一个 `struct epoll_event ev`，表示 `epoll` 修改红黑树时，需要关注的事件。 `ev` 除了事件之外，还有一个可以由用户定义的 `Union`，它可以是文件描述符，也可以是一个指针，这里我们就让他指向 `Channel` 类的一个对象，`Channel` 类中本来就有文件描述了，所以 `Union` 解释为指针，功能明显更强大。

```cpp
struct epoll_event {
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;

typedef union epoll_data {
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

```cpp
void Epoll::UpdateChannel(Channel *ch) {
    int fd = ch->getfd();
    struct epoll_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.data.ptr = ch; // 将 ev.data 解释为指向 channel 的指针；
    ev.events = ch->get_events();
    if (!ch->get_in_epoll()) {
        // 添加到 epoll 中
        errif(epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1, "epoll add error!\n");
        ch->set_in_epoll();
    } else {
        errif(epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev) == -1, "epoll mod error!\n");
    }
}
```

## Epoll 的变化

其他变化就是，`Epoll` 类中的 `Poll` 函数变成返回 `vector<Channel *>`，而不是返回 `epoll_event`。同时会设置 `ch->active_events`，表示该 `ch` 正在发生的事件。

## 主循环中

由于 `ep->Poll` 返回的是 `Channel *` 的集合，我们可以拿到 `ch` 对应的文件描述符和发生的事件，并对其进行处理。