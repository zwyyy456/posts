---
title: "MIT 6.S081 Page faults"
date: 2023-07-17T16:27:16+08:00
lastmod: 2023-07-17T16:27:16+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "mit", "os"]
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
## 概述

这一章主要聚焦于，我们利用 virtural memory 和 page fault 这两个机制，能够实现一些什么样的有意思的优化。

虚拟内存的有两大优势：

1. Isolation，保证每个进程都有它自己的虚拟地址空间，写自己的虚拟地址处的数据不会破坏其他进程的数据；
2. Levle of indirection，提供了一层抽象（这里不是很好理解），可以理解为提供了一层从虚拟地址到物理地址的映射关系，利用这个映射关系，我们可以实现很多有意思的优化。

利用 page fault，我们可以更新 page table，即更改虚拟地址和物理地址之间的映射关系（在之前的 xv6 中，可以说 va 和 pa 的映射关系，在进程启动之后，到进程结束之前，都是固定的）。

对于 page fault，也可以说是一种 trap，之前提到的 [system call](https://blog.zwyyy456.tech/zh/posts/tech/mit6.s081-lec06/) 是发生了系统调用之后的 trap，因此 trap 完成之后，我们会返回到产生系统调用的指令的下一条指令继续执行；而 page fault 则是异常（exception）导致的 trap，trap 结束之后，我们会返回导致 page fault 的指令，重新执行这一条指令；

正如 system call 导致的 trap 中，我们需要实现真正执行 systemcall 的函数；而 page fault 导致的 trap 中，我们也需要处理这一异常（一般是在 `trap.c` 的 `usertrap` 函数中）。

对于处理 page fautl 的思路，其实可以参照 system call，我们通过读取 scause 寄存器的值来判断导致 trap 的原因，如果是 $13$ 或者 $15$，则说明是 page fault。

然后，我们可以读取 stval 寄存器的值，找到发生 page fault 的虚拟地址 vaddr，而导致 page fault 的指令的地址，存放在 sepc 寄存器中。

![kt4jiXRA2bcJFOQ](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065952.jpg)

通过 page fault，我们可以实现一些非常有意思的优化方案，例如 lazy page allocation、zero fill on demand、copy on write fork、demand paging、memory mapped files 等；

## Lazy page allocation

调用 `sbrk` 时，如果参数 `n > 0`，那么只是单纯增加有效的虚拟地址的范围，从 `p->sz` 增加到 `p->sz + n`（即新的 `p->sz`），如果读取某个 **heap** 中的虚拟地址，发现该 va 没有映射到 pa 上，那么就会触发 page fault，分配物理页，让 va 映射到这个新的 PP 上。

具体细节可以参照这篇 [实验笔记](https://blog.zwyyy456.tech/zh/posts/tech/xv6-lab5/)。

这里尤其要注意一点，在内核态下，访问 user pagetable 下的未映射到 PP 的 vaddr，不会触发 page fault，因此在 `argaddr` 中，我们需要进行检查。

## Copy-on-Write fork

举个例子，当 shell 要执行一个命令时，shell 先 `fork` 一个子进程，`fork` 会为子进程创建一个父进程的拷贝，然后子进程或调用 `exec` 运行其他程序例如 `echo`，`exec` 要运行其他程序，要做的第一件事就是丢弃这个虚拟地址空间，转而使用一个包含了 `echo` 的新的地址空间。

因此，其中一个优化是，创建子进程时，不再“复制 PP，并将子进程的虚拟地址 map 到这个新的 PP”，而是将子进程的虚拟地址也 map 到父进程的这个 PP 上，即**父子进程共享 PP**。

为了防止子进程修改 VP 的内容时影响到父进程的 VP 的内容，或者父进程修改 VP 时影响到子进程的 VP 的内容（因为父子进程的 VP map 到了同一个 PP），我们需要将父进程和子进程的 VP 都设置为**不可写**的，并将 riscv 预留的 pte 标志位的第 $8$ 位标记为 $1$（PTE_C），当要写这个 VP 时，就触发 page fault，根据该虚拟地址是否映射了 PP 以及该虚拟地址的 PTE_W 位和 PTE_C 位，判断是否是写 COW page 导致的 page fault，至于如何处理，参照 [lab6: Copy-on-Write Fork for xv6](https://blog.zwyyy456.tech/zh/posts/tech/xv6-lab6/)。

## Zero fill on demand

在用户程序的地址空间中，除了 data、text 区域，还有一个 bss 区域，里面包含了未被初始化或者初始化为 $0$ 的全局或者静态变量，正常来说，程序执行的时候，是要为 bss 段的 VP，也都分配对应的物理页的，但是这其实并不必要。

例如，假设 bss 段有很多个 VP，并且基于 bss 段的定义，这里面的数据都是 $0$，那么我们不必为每一个 VP 都 map 单独的 PP，而是将 bss 段的所有的 VP 都 map 到一个值全为 $0$ 的 PP 上。

因此，我们需要将这些 VP 都设为不可写的，当要写其中一个 VP 时，策略与 Copy-on-Write fork 类似。

## Demanding paging

对于 `exec`，未修改的 xv6 中，os 会加载程序的 text、data 区域，并以 eager 的方式将这些区域加载进 page table（应该也可以说加载进物理内存），实际上我们可以采用 lazy 的方式，即为 text 和 data 分配好地址段，但是相应的 PTE 不对应任何 PP，即这些 PTE 的 valid bit 被设置为 $0$。

我们可以基于 accessed bit 实现 LRU 策略。