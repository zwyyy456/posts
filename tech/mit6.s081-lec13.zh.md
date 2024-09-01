---
title: "MIT 6.S081 Sleep & Wake up"
date: 2023-07-20T17:40:26+08:00
lastmod: 2023-07-20T17:40:26+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "mit", "os"]
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
## Sleep and wakeup

Sleep 允许一个内核线程等待某个特定事件的发生，另一个线程可以调用 `wakeup` 来表示这个正在等待时间发生的线程应该恢复了。

> Sleep and wakeup are often called sequence cooridination or conditional synchronization mechanisms.

xv6 中利用 `sleep` 和 `wakeup` 实现了一种 high-level 的同步机制，被称为信号量（semaphore），用于协调生产者和消费者（xv6 中并未使用信号量）。

> A semaphore maintains a count and provides two operations. The “V” operation (for the producer) increments the count. The “P” operation (for the consumer) waits until the count is non-zero, and then decrements it and returns.

```c
struct semaphore {
    struct spinlock lock;
    int count;
};
void V(struct semaphore *s) {
    acquire(&s->lock);
    s->count += 1;
    // wakeup(s); 
    release(&s->lock);
}
void P(struct *s) {
    while (s->count == 0) 
        //sleep(s);
        ;
    acquire(&s->lock);
    s->count -= 1;
    release(&s->count);
}
```

上述代码给出了一个非常简单但是性能不优秀的“生产者 - 消费者”模型实现，如果生产者很少工作，那么消费者会花费大量时间在 while 循环中。为了避免这一点，消费者应该要有办法主动让出 cpu，并且只在 V 递增 `s->count` 之后才恢复执行。

注释中给出了一种简单的实现办法。然而，仅仅是这样并不够，因为存在一个 **lost wake-up** 问题。

> （先考虑一个 P 和一个 V 的情况）假设 P 执行完 `while (s->count == 0)`，正准备执行 `sleep(s)`，这时候 V 递增了 `s->count`，然后准备唤醒一个线程，然而由于这时 P 还没有 sleep，因此没有线程可以被 V 唤醒，那么即使 `s->count` 已经不为 $0$ 了，P 还是执行了 `sleep(s)`，然而，除非 V 再次递增 `s->count`，否则 P 永远不会被唤醒。这就是 **lost wake-up**，即有一次 wake-up 丢失了，本该唤醒进程却没有，而 P 一直在等待这个已经发生了的唤醒。

如果简单的把 `acquire(&s->lock)` 移动到 P 的最开始，看起来可以解决问题，但是会造成更严重的死锁问题，P 持有锁，然后 `sleep`，因此 P 在被唤醒之前不会主动释放锁，而 V 需要持有锁才能唤醒 P。

为了解决 **lost wake-up** 问题，同时避免死锁，我们需要修改一下 `sleep` 的接口，

```c
void P(struct semaphore *s) {
    acquire(&s->lock);
    while(s->count == 0) 
        sleep(s, &s->lock)
    s->count -= 1;
    release(&s->lock);
}
```

在 sleep 中，我们会在 proc 已经被标记成 asleep 之后再释放 `s->lock`，这就保证了 V 直到 P 让自己完成睡眠之后才能尝试递增计数和唤醒进程。我们这里可以将 `s->lock` 称为 **condtion lock**。

## Code: Sleep and wakeup

xv6 的 `sleep` 和 `wakeup` 的实现机制与使用规则避免了 **lost wake-up**。

xv6 的 sleep 的基础机制就是 `sleep` 将当前进程标记为 `SLEEPING`，并调用 `sched` 来让出 cpu；而 `wakeup` 寻找等待队列中一个 sleeping 的进程，然后将它标记为 `RUNNABLE`。xv6 一般使用涉及 `wait` 的内核数据结构的地址作为 `channel`。

```c
void sleep(void *chan, struct spinlock *lk) {
    struct proc *p = myproc();
    acquire(&p->lock); // DOC: sleeplock1
    release(lk);
    // Go to sleep.
    p->chan = chan;
    p->state = SLEEPING;
    sched();
    // Tidy up.
    p->chan = 0;
    // Reacquire original lock.
    release(&p->lock);
    acquire(lk);
}
```

当要 `sleep` 的进程持有了 `&p->lock` 之后，就可以释放 `s->lock`（即 `lk` 了），持有 `lk` 是为了避免在进程进入睡眠之前，就有 V 来唤醒它，造成 **lost wake-up**，当持有 `&p->lock` 之后，由于 `wakeup` 进程需要持有该进程的锁，因此持有 `&p->lock` 就能避免该进程在进入睡眠之前就被唤醒了，于是我们可以释放 `lk` 了。而 `&p->lock` 会在完成睡眠，调用 `sched` 之后，由 scheduler 线程释放，这里跟定时器中断导致的进程切换是一致的。

理解了 `sleep`，再看 `wakeup` 其实就不难理解了。

```c
void wakeup(void *chan) {
    struct proc *p;
    for (p = proc; p < &proc[NPROC]; p++) {
        if (p != myproc()) {
            acquire(&p->lock);
            if (p->state == SLEEPING && p->chan == chan) {
                p->state = RUNNABLE;
            }
            release(&p->lock);
        }
    }
}
```

虚假唤醒：有时候可能会出现多个进程睡眠在同一个 `channel` 上的情况，例如多个进程从同一个 pipe 中读取数据，一次 `wakeup` 调用会唤醒所有的这些进程，但是当一个进程被 `scheduler` 线程选择，将数据读取完之后，递减计数，于是其他睡眠的进程会发现，自己虽然被唤醒了，但是 `s->count` 依旧是 $0$，这种情况我们被称为虚假唤醒，由于 `sleep` 处于 `while(s->count == 0)` 的循环中，因此会被困在循环中，再次睡眠，因此虚假唤醒这个问题是可以接受的。

> 这里 `sleep` 最后的 `acquire` 保证了，即使一次有多个睡眠进程被标记为 RUNNABLE，也只有一个进程能够获取到 `lk`，执行生产者的操作并递减 `s->count`。

`sleep` 与 `wakeup` 的吸引力在于它们都非常轻量，不需要创建特定数据结构来充当 sleep 的 channel，并且提供了一层间接性，调用者不需要知道具体与哪个进程交互。

## Code: Pipes

这里要主要是理解 `piperead` 和 `pipewrite`，这要求我们注意两点：一是 `sleep` 会申请进程锁再释放 `pi->lock`，scheduler 线程切换回 `sleep` 之后，会释放进程锁再申请 `pi->lock`。而 `wakeup` 只会申请与释放进程锁。

`piperead` 中 `pi->nread == pi->nwrite` 就起到了 `s->count == 0` 的作用，在 `pipewrite` 中则是 `pipewrite == piperead + PIPESIZE`，`pipewrite` 是生产者， `piperead` 是消费者。

pipe 的实现针对 reader 和 writer 使用了不同的 channel（`&pi->nread` 与 `&pi->nwrite`）。

```c
int pipewrite(struct pipe *pi, uint64 addr, int n) {
    int i = 0;
    struct proc *pr = myproc();

    acquire(&pi->lock);
    while (i < n) {
        if (pi->readopen == 0 || pr->killed) {
            release(&pi->lock);
            return -1;
        }
        if (pi->nwrite == pi->nread + PIPESIZE) { // DOC: pipewrite-full
            wakeup(&pi->nread);
            sleep(&pi->nwrite, &pi->lock);
        } else {
            char ch;
            if (copyin(pr->pagetable, &ch, addr + i, 1) == -1)
                break;
            pi->data[pi->nwrite++ % PIPESIZE] = ch;
            i++;
        }
    }
    wakeup(&pi->nread);
    release(&pi->lock);

    return i;
}

int piperead(struct pipe *pi, uint64 addr, int n) {
    int i;
    struct proc *pr = myproc();
    char ch;

    acquire(&pi->lock);
    while (pi->nread == pi->nwrite && pi->writeopen) { // DOC: pipe-empty
        if (pr->killed) {
            release(&pi->lock);
            return -1;
        }
        sleep(&pi->nread, &pi->lock); // DOC: piperead-sleep
    }
    for (i = 0; i < n; i++) { // DOC: piperead-copy
        if (pi->nread == pi->nwrite)
            break;
        ch = pi->data[pi->nread++ % PIPESIZE];
        if (copyout(pr->pagetable, addr + i, &ch, 1) == -1)
            break;
    }
    wakeup(&pi->nwrite); // DOC: piperead-wakeup
    release(&pi->lock);
    return i;
}
```

## Code: wait, exit and kill

`wait` 首先会申请 condtion lock，即 `wait_lock`，保证 `wait` 不会错过 `exit` 的 `wakeup`，然后寻找子进程中的 ZOMBIE 进程，释放其资源和 `proc` 结构体，拷贝子进程的 exit status，释放锁，并返回子进程的 pid。如果 `wait` 发现没有子进程退出，就会调用 `sleep` 来等待子进程退出。被退出的子进程调用 `wakeup` 唤醒并由 `scheduler` 线程返回到 `sleep` 的下一条语句之后，它会继续执行查找 ZOMBIE 子进程并释放其资源的任务。

假设进程 a 调用 `exit`，那么`exit` 记录了 exit status，释放进程 a 的一些资源，调用 `reparent` 把 a 的子进程的转让给 `init` 进程，即进程 a 的子进程的父进程，会变成 `init` 进程。`exit` 还会将进程 a 的状态标记为 ZOMBIE，永远让出 cpu（毕竟 scheduler 线程只会选择 RUNNABLE 进程）。`exit` 会持有 `wait_lock` 和 `p->lock`，`wait_lock` 是生产者 `exit` 和消费者 `wait` 的 condtion lock，而持有 `&p->lock` 是为了防止 `wait` 在 `exit` 调用 `swtch` 之前就观察到该进程是 ZOMBIE 的了。**为了防止死锁，`exit` 与 `wait` 申请锁的顺序是相同的**。

`kill` 则只是将进程 `p->killed` 标记为 $1$，将进程状态从 SLEEPING 修改为 RUNNABLE，从而让进程能被唤醒，然后回到 `usertrap.c` 调用 `exit` 自杀，如果 victim 进程本身是在 user space 中运行的，那么它很快会因为定时器中断进入 `usertrap` 然后调用 `exit`。

`kill` 的将状态从 SLEEPING 修改为 RUNNABLE 可能带来隐患，睡眠中的进程可能没等到它需要的条件就被唤醒了，因此 `while` 循环中还需要判断 `p->killed` 的状态，如果被设置了，那么就放弃当前的活动，例如 pipe。

也有一些 xv6 的 `sleep` 所处的 `while` 循环不检查 `p->killed`，因为代码应该是原子操作的多步系统调用的中间，例如 `virtio` 驱动程序，它不检查 `p->killed`，等待磁盘 I/O 时被杀死的进程将不会退出，直到它完成当前系统调用并且 `usertrap` 看到了 `p->killed` 标志。

## Process Locking

进程的锁（`p->lock`）应该是 xv6 中最复杂的锁了。在进程要读写 `struct proc` 的 `p->state, p->chan, p->killed, p->xstate, p->pid` 等字段时，都必须持有 `p->lock`，因为这些字段可能被其他进程，或者其他 cpu 上的调度器线程访问。

`p->lock` 做了以下事情：

- 与 `p->state` 一起，避免从 `proc[]` 数组创建新线程时的 race condition；
- 当进程被创建或者销毁时，避免它被别的进程看到；
- 避免父进程在 `wait` 循环中看到一个状态为 ZOMBIE 但是还没有让出 cpu 的子进程；
- 防止其他 cpu 的 scheduler 线程去运行一个马上要让出 cpu，已经将状态设置为 RUNNABLE 但是还没有结束 `swtch` 调用的进程；
- 保证一个 RUNNABLE 的进程只会被一个 scheduler 线程运行；
- 防止进程在调用 `swtch` 中时，被定时器中断再驱使着去执行 `yield`；
- 防止 `wakeup` 进程看到有状态处于 SLEEP 但是还没有完成让出 cpu 的动作的进程；
- 使得 `kill` 的检查 `p->pid` 和设置 `p->killed` 原子化；
- It prevents the victim process of `kill` from exiting and perhaps being re-allocated between `kill`’s check of `p->pid` and setting `p->killed`.
    - 假设没有这个 `p->lock` 的话，它可能检查完 `p->pid` 之后，这个调用 `killed` 的进程就被暂停了，然后让出了 cpu，这时一个新进程被分配，并且二者具有相同的 pid，因此导致新进程受到影响，而不是影响原来的目标进程。
