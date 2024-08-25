---
title: "Web Day11：完成线程池以及加入一个简单的测试程序"
date: 2023-11-18T19:40:26+08:00
lastmod: 2023-11-18T19:40:26+08:00 #更新时间
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
## 需要完善的地方

Day10 中，我们添加了一个简单的线程池，一个完整的 Reactor 模型已经成型。但这个线程池存在的问题还比较多，例如任务队列的取出、添加都存在拷贝，性能较差，只能用于学习。

正确操作应该使用右值移动、完美转发等来阻止拷贝。

另外，线程池只能接受 `std::function<void()>` 类型的函数，所以函数需要使用 Lambda 表达式来创建，或者 `std::bind()`，且无法得到返回值。

## 利用模板

```cpp
class ThreadPool {
  private:
    std::vector<std::thread> threads_;
    std::queue<std::function<void()>> tasks_;
    std::mutex tasks_mtx_;
    std::condition_variable cv_;
    bool stop_;

  public:
    explicit ThreadPool(int size = 8);
    ~ThreadPool();

    template <class F, class... Args>
    auto add_task(F &&f, Args &&...args) -> std::future<typename std::result_of<F(Args...)>::type>;
};

template <class F, class... Args>
auto ThreadPool::add_task(F &&f, Args &&...args) -> std::future<typename std::result_of<F(Args...)>::type> {
    using return_type = typename std::result_of<F(Args...)>::type;
    auto ptask = std::make_shared<std::packaged_task<return_type()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...));
    /* auto ptask = std::make_shared<std::packaged_task<return_type()>>([f = std::forward<F>(f), args = std::make_tuple(std::forward<Args>(args)...)]() mutable { */
    /*     return std::apply(std::move(f), std::move(args)); */
    /* }); */
    std::future<return_type> res = ptask->get_future();
    {
        std::unique_lock<std::mutex> lock(tasks_mtx_);
        // don't allow enqueueing after stopping the pool
        if (stop_) {
            throw std::runtime_error("enqueue on stopped ThreadPool");
        }
        tasks_.emplace([ptask]() { (*ptask)(); });
    }
    cv_.notify_one();
    return res;
}
```

## 代码说明

在函数声明的部分有 

```cpp
template <class F, class... Args>
auto add_task(F &&f, Args &&...args) -> std::future<typename std::result_of<F(Args...)>::type>;
```

这里谈谈我个人的一点理解，首先，是 `template<class F, class... Args>`，这是函数模板，表示函数可以接受两种任意类型的参数，一个参数类型是暂定是 F，到编译阶段才会被推导出来。另一个参数则是可变模板参数，你可以理解为它可以接受多个不同类型的参数。

因此，单看 `template<class F, class... Args>`，似乎和 `template<class... Args>` 没有区别。

然而，后面的 `std::future<typename std::result_of<F(Args...)>::type>` 则限定了 `F` 的类型一定是 callable 类型，后面简称函数类型。

```cpp
void EventLoop::add_thread(std::function<void()> func) {
    thread_pool_->add_task(func);
}
```

结合 `add_task` 的函数声明的要求，我们在通过调用 `add_task` 来实现 `add_thread` 的时候，只要求传递给 `add_task` 的第一个参数，必须是可调用类型，即 `func`，后面的参数可以再自行决定。

`std::future<typename std::result_of<F(Args...)>::type>` 使用了 `std::future` 库，`typename` 用于指示编译器 `std::result_of<F(Args...)>::type` 是一个类型的关键字。

`std::result_of` 是一个类型推导表达式。用于确定调用类型为 F 的可调用对象（比如函数、函数指针、Lambda等）时，使用参数类型为 Args... 的参数列表所返回的类型。

`std::future<typename std::result_of<F(Args...)>::type>` 通常用于异步编程中提交任务并且获取结果。

```cpp
using return_type = typename std::result_of<F(Args...)>::type;
auto ptask = std::make_shared<std::packaged_task<return_type()>>(
    std::bind(std::forward<F>(f), std::forward<Args>(args)...));
```

注意 `std::make_shared<T>(x)` 中，x 是用来构造 T 类型的对象的参数。

`std::packaged_task` 是 C++ 标准库中的一个模板类，它表示可以异步执行的任务，并返回一个值。

> `std::packaged_task<return_type()>`` 中 `return_type`` 表示的是一个类型，为什么后面还跟着一个括号？
> 这里的括号是 C++ 中用于表示返回值类型的语法，`<return_type()>` 表示将 `return_type` 视为一个类型，表示 `std::packaged_task` 包装的任务将返回这个类型的值。

例如：

```cpp
std::packaged_task<int()> task([]() {
    return 42;
}); // 希望 std::packaged_task 包装的任务返回一个整数类型的结果
```

## 添加测试程序

简单来说就是创建 n 个线程，每个线程发送接受相同的消息 k 次。

`./test -t 10000 -m 10 -w 5`

## Acceptor 的一点改进

对于 Acceptor，接受连接的处理时间短、报文数据小，并且同一时间一般不会有特别多的新连接到来。所以 Acceptor 没必要采用 ET 模式，也没有必要采用线程池。

即然 Acceptor 不会成为性能瓶颈，那么最好使用阻塞式 socket。

所以，day11 的源码中做了以下改变：

1. Acceptor 的 serv_sock 采用不设置成非阻塞；
2. Acceptor 使用 LT 模式，Connection 使用 ET；
3. Acceptor 建立连接不使用线程池，建立好连接之后，处理事件使用线程池；
