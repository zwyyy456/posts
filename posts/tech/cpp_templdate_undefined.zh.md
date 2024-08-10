---
title: "C++ 模板类编译过程中出现“undefined reference to”问题"
date: 2023-03-14T18:35:39+08:00
lastmod: 2023-03-14T18:35:39+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cpp"]
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
## 问题描述
C++在使用模板(template)类的时候，如果将类的成员函数的声明和实现分别放在`.h`头文件和`.cpp`源文件中，编译时会报错`undefined reference xxx`，找不到对应成员函数。

## 起因
`.h`文件中类的声明为：
```cpp
// 线程池，定义成模板类，为了代码的复用
template <typename T>
class ThreadPool {
    ...
  public:
    bool append(T *request);
    ...
};
```

`.cpp`文件中成员函数的实现为：
```cpp
template <typename T>
bool ThreadPool<T>::append(T *request) {
    // 操作工作队列时一定要加锁，因为它被所有线程共享
    queue_locker_.lock();
    if (work_queue_.size() > max_requests_) {
        queue_locker_.unlock();
        return false;
    }

    work_queue_.push_back(request);
    queue_locker_.unlock();
    queue_sta_.post();
    return true;
}
```

直接使用g++编译，会报错：
![atMelNiX1Zn3EKj](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065823.png)

## 原因分析
`template`其实是一种类似语法糖的东西，C++中每一个对象所占用的空间大小，是在编译的时候就确定的，在模板类没有真正的被使用之前，编译器是无法知道，模板类中使用模板类型的对象的所占用的空间的大小的。只有模板被真正使用的时候，编译器才知道，模板套用的是什么类型，应该分配多少空间。这也就是模板类为什么只是称之为**模板**，而不是**泛型**的缘故。

即`ThreadPool<int>`和`Thread<HttpConn>`是两个不同的类型，其成员函数也是两个不同的成员函数。

在编译`thread_pool.cpp`时，编译器会去查找对类`Thread<HttpConn>`的声明，如果找不到这个声明，那么就报错了。

## 解决方案
在成员函数的实现的代码所在的源文件的开头，声明该类，即添加：
```cpp
template class ThreadPool<HttpConn>;
```

或者将函数的实现也写在头文件中（不推荐）。


