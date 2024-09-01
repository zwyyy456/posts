---
title: "Xv6 Lab7: Multithreading"
date: 2023-07-22T11:30:01+08:00
lastmod: 2023-07-22T11:30:01+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "os", "lab", "mit"]
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
## Uthread: switching between threads

这个题还是对的起它 moderate 的难度了，如果认真看了 book-riscv-rev2.pdf 的 Scheduling 章节，以及看了这个 [课程翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec11-thread-switching-robert)，那么这题可以很快做出来，个人觉得 pdf 讲得更加清楚一些。

这个题甚至帮你把需要添加代码的地方都标注出来了，参照题目说明，主要有三步：

1. 修改 `thread_create` 来保证当 `thread_schedule` 第一次运行 `thread_create` 创建出来的线程时，该线程就会在自己的 stack 上执行传递给 `thread_create` 的函数，这里我们可以参照 `allocproc` 的实现，在 `thread_create` 标记出来的要我们添加代码的地方添加如下三行：

```c
memset(&t->context, 0, sizeof(t->context));
t->context.ra = (uint64)func;
t->context.sp = (uint64)t->stack + STACK_SIZE;
```

2. 保证 `thread_switch` 会切换并保存寄存器，这里参照 `scheduler` 的实现即可，在注释标记的地方添加以下语句，并且在 `uthread_switch.S` 中实现 `thread_switch` 函数（照抄 `swtch` 即可）：

```c
thread_switch((uint64)&t->context, (uint64)&current_thread->context);
```

```asm
thread_switch:
	/* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	    ret    /* return to ra */
```

3. 修改 `strcut thread` 来存储 `thread_switch` 时需要保存的寄存器，还是参照 `struct proc` 即可：

```c
struct t_context {
    uint64 ra;
    uint64 sp;

    // callee saved
    uint64 s0;
    uint64 s1;
    uint64 s2;
    uint64 s3;
    uint64 s4;
    uint64 s5;
    uint64 s6;
    uint64 s7;
    uint64 s8;
    uint64 s9;
    uint64 s10;
    uint64 s11;
};

struct thread {
    char stack[STACK_SIZE]; /* the thread's stack */
    int state;              /* FREE, RUNNING, RUNNABLE */
    struct t_context context;
};
```

这样修改之后就能通过 `uthread` 了。

## Using threads

首先，注意 `include <pthread.h>` 应该在文件内容的最前面，否则 `make ph` 时容易出现意料之外的问题。

在多线程执行 `ph` 时，之所以会出现 xxx keys missing，是因为写的时候，假设有两个 put 进程，keys 的数量一共为 $100$，那么进程 $1$ 会负责写 $0\sim 49$，进程 $2$ 会负责写 $50\sim 99$，`NBUCKET = 5`，那么两个进程会往同一个 bucket 中写入 entry，这就是出现问题的原因。

假设进程 $1$ 执行 `insert(1, 3, &table[1], table[1]);`，进程 $2$ 执行 `insert(6, 2, &table[1], table[1])`，就会出现类似 `kalloc` 的 freelist 中，两个进程同时往链表插入头结点的情况。

解决方案很简单，针对要访问的 `table[i]` 加上互斥锁来保护即可，因此，我们需要一个互斥锁的数组 `mutex`，数组大小为 `NBUCKET`，需要访问 `table[i]`，就申请持有 `mutex[i]` 。

读取的时候不需要上锁，记得要先在 `main` 函数中初始化互斥锁。

## Barrier

这个其实不难，只不过之前的 lab 让我有点害怕 moderate。。。

主要是要防止第二次 round 不会影响上一次的 round。我们保证每次，所有的线程都到达 barrier 之后才会增加一次 `bstate.round`。

```c
static void barrier() {
    // YOUR CODE HERE
    //
    // Block until all threads have called barrier() and
    // then increment bstate.round.
    //
    // nthread 就是我们输入的第二个参数
    pthread_mutex_lock(&bstate.barrier_mutex);
    ++bstate.nthread;
    if (bstate.nthread == nthread) {
        bstate.round += 1;
        bstate.nthread = 0;
        pthread_cond_broadcast(&bstate.barrier_cond);
    } else {
        pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }
    pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

其实一开始因为想着条件变量总会配合 `while` 循环，因此写了两个 `while` 循环，两个条件变量来实现，但是按上面的单纯 `if` 就可以实现了。

