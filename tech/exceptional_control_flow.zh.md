---
title: "异常控制流（Exceptional Control Flow, ECF）"
date: 2023-06-27T15:41:06+08:00
lastmod: 2023-06-27T15:41:06+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["csapp"]
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
## 异常控制流简介

从给处理器上电起，直到断电，程序计数器（PC）假设一个值的序列：
$$ a_0, a_1, \cdots, a_{n - 1}$$

每个 $a_k$ 是某个相应的指令 $I_k$ 的地址，每次从 $a_k$ 到 $a_{k + 1}$ 的过渡称为控制转移（control transfer）。这样的控制转移序列叫做处理器的控制流（flow of control 或 control flow）。

最简单的控制流是一个平滑的序列，即 $I_k$ 和 $I_{k + 1}$ 在内存中是连续的。

现代系统通过使控制流发生突变来应对系统状态的变化。一般而言我们把这些突变称为**异常控制流（Exceptional Control Flow, ECF）**。

例如当前程序在执行地址 $a_k$ 对应的指令 $I_k$，正常情况下，下一个指令应该是应该是地址 $a_{k + 1}$ 对应的指令 $I_{a_{k + 1}}$，但是由于发生了 page fault，它转去执行内核态的**缺页异常处理程序**，对应指令 $I_j$，执行完 $I_j$ 之后，它又回来执行 $I_k$。

> For example, at the hardware level, events detected by the hardware trigger abrupt control transfers to exception handlers. At the operating systems level, the kernel transfers control from one user process to another via context switches. At the application level, a process can send a signal to another process that abruptly transfers control to a signal handler in the recipient.


ECF 与进程管理直接相关

## 异常（Exception）

系统给所有可能的每种类型的异常都分配了一个唯一的非负整数的异常号（exception number）。其中一些号码是由处理器的设计者分配的，其它号码则是由操作系统内核（操作系统常驻内存的部分）的设计者分配的。

- CPU 设计者分配的：devided by zero、page fault、断点、算术运算溢出、内存访问非法；
- 内核设计者分配额：系统调用和来自外部 I/O 设备的信号。

在系统启动（计算机重启或者上电）时，操作系统分配和初始化一张被称为异常表（exception table）的跳转表，使得 table entry $k$ 包含异常 $k$ 的处理程序的地址。

异常表的起始地址存放在一个叫做异常表基址寄存器（exception table base register）的特殊 CPU 寄存器中，异常号 $k$ 可以看作是异常表的索引。

![rHpD8J2AX7OQ9xb](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065820.png)

异常可以分为四类：

### 中断（interrupt）

中断是异步发生的，是来自处理器外部的 I/O 设备的信号的结果。

> 硬件中断不是由任何一条专门的指令造成的，从这个意义上说它是异步的。硬件中断的异常处理程序常常被称为**中断处理程序（interrupt handler）**。

### 陷阱（trap）

### 故障（fault）

### 终止（abort）

