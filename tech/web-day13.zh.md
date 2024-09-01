---
title: "Web Day13：服务器完善"
date: 2023-11-20T10:45:06+08:00
lastmod: 2023-11-20T10:45:06+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: [""]
tags: [""]
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

## 定时器定时关闭连接

首先，应该每个 EventLoop 包含一个 Timer，Timer 的主体结构是一个小顶堆，`priority_queue<pair<double, Connection *>>`，double 表示以微秒计数的超时时间 t，即当前时间如果大于 t，那么就超时了。

同时 Connection 需要额外添加一个变量，即 `cnt`，标志该 Connection 在小顶堆中的数量；

channel 的回调函数中，需要将 Connection 的次数 + 1，同时再 push 这个 Connection 到小顶堆中。

每次 Loop 循环完成，就获取当前时间，进行一次 pop，直到堆顶的的元素的超时时间大于当前时间。

每次 pop 的时候，被 pop 的元素的 cnt 减一，如果减到 0 了，那么就调用 `delete_conn_callback` 函数。

由于每个 EventLoop 只由一个特定的线程负责，因此小顶堆不需要加锁来保护。

## 日志系统

同步日志：产生日志的同时就将其写入至文件中，即在 EventLoop 中，该 EventLoop 线程直接负责写入日志到文件（磁盘）。写日志的过程中，由于要写入磁盘，耗时比内存中的 IO 要长，很可能影响到服务器的效率。

异步日志：采用一个单独的线程，而非 EventLoop 线程来向磁盘写入日志，可以说是一个典型的生产者消费者模型。

分为日志前端和日志后端，前端就是生产者，也就是 I/O 线程这些，负责将日志写入到位于内存的 log_buffer 中，而消费者则是后端，负责将日志从 log_buffer 写入到磁盘中去。

前端线程会调用 current_buffer_->append，写到 AsyncLogging 类的 写入到 current_buffer_，如果 current_buffer_ 满了，就调用 `buffers_.push_back(std::move(currentBuffer_));` 移动到 buffer 队列中去。

然后 `currentBuffer_ = std::move(nextBuffer_);`，即把 `nextBuffer_` 重用为 `currentBuffer_`。

后端线程函数threadFunc，会构建1个LogFile对象，用于控制log文件创建、写日志数据，创建2个空闲缓冲区 `buffer1`、`buffer2`，和一个待写缓冲队列 `buffersToWrite`，分别用于替换当前缓冲currentBuffer_、空闲缓冲nextBuffer_、已满缓冲队列buffers_，避免在写文件过程中，锁住缓冲和队列，导致前端无法写数据到后端缓冲。

threadFunc中，提供了一个loop，基本流程是这样的：
1）每次当已满缓冲队列中有数据时，或者即使没有数据但3秒超时，就将当前缓冲加入到已满缓冲队列（即使当前缓冲没满），将buffer1移动给当前缓冲，buffer2移动给空闲缓冲（如果空闲缓冲已移动的话）。
2）然后，再交换已满缓冲队列和待写缓冲队列，这样已满缓冲队列就为空，待写缓冲队列就有数据了。
3）接着，将待写缓冲队列的所有缓冲通过LogFile对象，写入log文件。
4）此时，待写缓冲队列中的缓冲，已经全部写到LogFile指定的文件中（也可能在内核缓冲中），擦除多余缓冲，只用保留两个，归还给buffer1和buffer2。
5）此时，待写缓冲队列中的缓冲没有任何用处，直接clear即可。
6）将内核高速缓存中的数据flush到磁盘，防止意外情况造成数据丢失。