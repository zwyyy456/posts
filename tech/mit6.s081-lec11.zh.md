---
title: "MIT 6.S081 Thread switching"
date: 2023-07-19T10:46:24+08:00
lastmod: 2023-07-19T10:46:24+08:00 #更新时间
authors: ["zwyyy456"] #作者
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
## Multiplexing

xv6 通过将 cpu 从一个进程切换到另一个进程来实现 multiplex（多路复用），进程的切换会在两种情形下发生：

1. xv6 的 sleep 与 wakeup 机制在进程等待 IO 完成或者等待子进程退出又或者在 `sleep` 系统调用中等待的时候切换进程。
2. xv6 会周期性地强制切换进程，从而应对那些长时间切换而未 `sleep` 的进程。

这个 multiplex 机制会让进程产生一种自己完全拥有 cpu 的错觉，就像 xv6 用虚拟内存和 page table 机制让进程觉得自己拥有完整的内存空间一样。

xv6 使用硬件定时器中断来保证 context switch（上下文切换）。

## Code: Context switching

用户进程之间的切换步骤如下图所示：

![b41nAVjQsSLyDBv](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065912.png)

用户进程之间的切换其实会经过两次 context switch，以上图为例，第一次是从 shell 用户进程的 kernel thread 切换到 cpu 的 scheduler thread；第二次从 cpu 的 scheduler thread 切换到新用户进程（例如 cat）的 kernel thread。

> 在 xv6 中，我们可以认为每个用户进程，包含一个内核线程与一个用户线程，然后每个 cpu 包含一个 scheduler thread，schedular thread 工作在内核中，有只属于它的 kernel stack。

`swtch` 执行为内核线程切换的保存和恢复工作。`swtch` 的主要工作就是保存和恢复 riscv 的寄存器，又被称为上下文。

当一个进程要让出 cpu 时，该进程的内核线程会调用 `swtch` 来把该进程的 context 保存起来，并返回到 scheduler 线程的 context，每个 context 都包含于 `strcut context`。 `context` 结构体可能是 `struct proc` 的成员或者 `struct cpu` 的成员（ scheduler 线程）。

```c
struct context {
  uint64 ra;
  uint64 sp;
  // callee-saved
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
```

`swtch` 函数实际上是利用汇编实现的，它接受两个参数 `struct context *old` 和 `struct context *new`，将当前寄存器保存在 `old`，中然后从 `new` 中加载内容到当前寄存器，然后 `return`。

用户进程要让出 cpu，切换到 scheduler 线程，会经历 `usertrap => yield => sched => swtch` 的过程，`swtch` 把当前进程的上下文保存在 `p->context` 中，从 `cpu->context` 加载当前 cpu 的 scheduler 线程的 context。

`swtch` 只会保存 callee-saved 寄存器（这里 `sched` 是 caller， `swtch` 是 callee）。`swtch` 不保存 pc，而是保存 ra 寄存器（ra 寄存器存放了 `swtch` 调用结束后应该返回到的地址，可以理解为 `swtch` 语句的下一条语句）。

scheduler 的 context 是在 scheduler 线程调用 `swtch` 的时候被保存在 `cpu->context` 的。

进程切换的流程可以这么理解，进程 a 调用 `swtch`，将进程 a 的 context 保存在 `proca->context`（包括进程 a 的 ra 寄存器），从 `cpu->context` 中加载 context，由于 context 中包含了 ra 寄存器，而 `swtch` 函数的最后一条指令就是 `ret`，因此会跳转到 ra 寄存器的地址处继续执行，这里应该就是执行 `scheduler` 函数中的 `c->proc = 0`，由于 `scheduler` 本身是个死循环，`c->proc = 0` 的下一个语句就是一个新的 `for` 循环，就又会执行到 `swtch`，将 scheduler 线程的 context 保存在 `cpu->context` 中，然后从 `procb->context` 中加载 context（包括进程 b 的 ra 寄存器），然后跳转到进程 b 的 ra 寄存器的地址处继续执行（这里类比进程 a，就是进程 b 调用的 `swtch` 语句的下一条语句），这里从结果来看，就是让进程 b 会从系统调用或者中断响应中退出，继续执行它的本职工作。

## Code: Scheduling

`scheduler` 的实现如下：

```c
void scheduler(void) {
    struct proc *p;
    struct cpu *c = mycpu();

    c->proc = 0;
    for (;;) {
        // Avoid deadlock by ensuring that devices can interrupt.
        intr_on();

        for (p = proc; p < &proc[NPROC]; p++) {
            acquire(&p->lock);
            if (p->state == RUNNABLE) {
                // Switch to chosen process.  It is the process's job
                // to release its lock and then reacquire it
                // before jumping back to us.
                p->state = RUNNING;
                c->proc = p;
                swtch(&c->context, &p->context);

                // Process is done running for now.
                // It should have changed its p->state before coming back.
                c->proc = 0;
            }
            release(&p->lock);
        }
    }
}
```

我们可以发现，在调用 `swtch` 时，xv6 持有 `&p->lock`，但是这个锁，会在 `scheduler` 线程中马上被解锁。

我们考虑一个进程 a 切换到进程 b 在切换到进程 c 的过程，就能弄明白这个过程：

进程 a 调用 `yield`，此时 `pa->context` 保存的 ra 就是 `release(&pa->lock)`，然后进入到 scheduler 线程，由于 `cpu->context` 保存的 ra 就是 `c->proc = 0; release(&px->lock);`（这个 x 到底是哪个进程，也许现在还想不明白，但是很快就能意识到，x 其实就是 a），所以马上执行 `c->proc = 0; release(&px->lock);`，然后选择了进程 b，于是我们给 b 加锁，然后执行 `swtch`，这里就知道，`cpu->context` 的 ra 中会保存 `c->proc = 0; release(&pb->lock);`；然后，就执行到 `pb->context` 的 ra 对应的指令了。当进程 b 因为定时器中断，要让出 cpu 时，进程 b 调用 `yield`，然后切换到 scheduler 线程时，我们就知道此时的 ra 寄存器，即 `cpu->context` 的 ra，就是 `c->proc = 0; release(&pb->lock)`，即我们会马上释放进程 b 的锁。

- 其实就是要意识到一点，进程 a 让出 cpu 之前，必然是被另一个进程 x 切换到了进程 a，因此从进程 a 切换到 scheduler 线程，会马上释放进程 a 的锁！

从考虑上面的进程 a 切换到 进程 b 的过程，我们发现，回到进程 b 的 `sched` 语句的下一条语句，就是 `release`，但是这里释放的进程 b 的锁，其实应该是在 scheduler 线程中被申请的。

彼此之间有意识地通过线程传递 cpu 控制权的进程有时候也被称作**协程（coroutines）**，`sched` 和 `scheduler` 就可以看作是彼此的协程。

> 只有一种情况，scheduler 调用 `swtch` 不会返回到要切换到的进程的 `sched();` 语句的下一条指令处，那就是创建（子）进程的时候，allocproc 会将进程的 ra 设置为 `forkret` 函数，这样哪怕子进程之前没有调用过 `swtch`，也能够被切换到这个子进程，这样会先释放子进程的锁，再执行 `usertrapret` 返回到子进程的用户空间，否则可能子进程会直接返回到 `usertrapret` 再返回用户空间，子进程的锁不会被释放。