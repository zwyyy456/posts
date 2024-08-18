---
title: "MIT 6.S081 Multiprocessors and locking"
date: 2023-07-18T13:54:16+08:00
lastmod: 2023-07-18T13:54:16+08:00 #更新时间
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
## why lock

防止多核并行运行下的 race condition 导致的错误。

内核中的数据是典型的 concurrently-accessed 的数据。

## race condition and how the lock avoid it

A race condition is a situation in which a memory location is accessed concurrently, and at least one access is a **write**. 

Locks ensure **mutual exclusion**. 锁的持有与释放之间的语句会被**原子化**，在 xv6 中，就是 `acquire` 和 `release` 之间的多条语句，只能被一个进程（CPU）全部执行完了之后，才可能被其他的进程（CPU）执行，这样就避免了 race condition。

`acquire` 和 `release` 之间的多条指令通常被称为 **critical section**。

lock 在某种意义上是在维护 critical section 中一些数据的不变量（some collection of invariants），这个不变量在 critical section 中可能会被暂时破坏，但是当 critical section 的最开始，以及结束的时候，这个 invariants 一定成立！
例如 `kfree` 中的 lock，就是在维护 `kmem.freelist` 一定指向 freelist 的头结点这一不变量。

内核设计的一大主要挑战就是避免**锁争用（lock contention）**。

## Code: Locks

xv6 有 spinlocks（自旋锁）和 sleep-locks 两种

spinlock 的实现如下：

```c
// Mutual exclusion lock.
struct spinlock {
    uint locked; // Is the lock held?

    // For debugging:
    char *name;      // Name of lock.
    struct cpu *cpu; // The cpu holding the lock.
};
```

spinlock 的关键部分在于 `locked` 字段，`locked = 1` 时，说明 lock 已经被持有了，`locked = 0` 时，则说明 lock 未被持有，即 cpu 可以申请持有该锁。

`acquire` 一个简单但是错误的实现如下：

```c
void acquire(struct spinlock *lk) {
    for (;;) {
        if (lk->locked == 0) {
            lk->locked = 1;
            break;
        }
    }
}
```

这个实现的问题在与，假设有 cpu1 和 cpu2，cpu1 先观察到 `lk->locked == 0` 了，正准备执行 `lk->locked = 1` 时，cpu2 也观察到了 `lk->locked == 0`，那么就存在两个 cpu 同时拿到锁的情况了。因此，我们必须保证 `if (lk->locked == 0)` 和 `lk->locked = 1` 这两条语句被原子（atomic）地执行。即不存在 cpu1 执行完语句 $1$，还没执行语句 $2$ 时就被 cpu1 执行语句 $1$ 的情况。

多核处理器一般会提供原子指令保证这两条语句原子化地执行，在 riscv 中，有 `amoswap r, a`。该指令读取地址 `a` 处的值，记为 `tmp`，然后将寄存器 r 中的值写入 `a`，再将 `tmp` 的值写入寄存器 r。该指令实际上就是原子地执行交换地址 `a` 和寄存器 `r` 处的值。

所以，`acquire` 的实现要想不出错，就应该执行原子地交换 `val`（值为 $1$） 与 `l->locked` 的值，然后检查 `val` 的值，如果是 $0$，就说明获取到了锁，否则说明没有获取到，并且执行交换后，`l->locked` 一定是 $1$。由于执行原子交换，也不可能存在两个 cpu 同时检查到 `val` 为 $0$ 的情况。

![ZfPwFynJepvqgQh](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070009.png)

> It performs this sequence atomically, using special hardware to prevent any other CPU from using the memory address between the read and the write.

xv6 的 `acquire` 函数调用了 portable C library 的 `__sync_lock_test_and_set` 函数，本质上就是上面提到的 `amoswap` 函数。

xv6 的 `release` 函数调用 `__sync_lock_release(&lk->lock)`，该函数执行原子地将 `lk` 的 `locked` 字段写为 $0$。

为什么不能使用一个普通的写入语句将 `lk->locked` 的值写为 $0$ 呢？这是因为：普通的赋值语句，riscv 中对应的汇编语句是 `store`，而 `store` 本身并不是原子的（可能导致缓存一致性失效？，这里不使用原子指令的副作用感觉不是很清晰 TODO(zwyyy)）。

> 由于每个 cpu 都有自己的 cache，将内存中的某个值写为 $0$，可能要经过先将 $cache$ 中的对应值写为 $0$，再写回内存的过程， 因此 store 指令不是原子的。

编译器在编译代码时，可能会重排指令以获取更好的性能，例如：

```c
acquire(&lock);
++i;
release(&lock);
```

编译器可能重排指令导致 `++i` 语句优先于 `acquire` 指令，这就违背了我们的初衷。 xv6 的源码中的 `acquire` 函数的实现如下：

```c
void acquire(struct spinlock *lk) {
    push_off(); // disable interrupts to avoid deadlock.
    if (holding(lk))
        panic("acquire");
    // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
    //   a5 = 1
    //   s1 = &lk->locked
    //   amoswap.w.aq a5, a5, (s1)
    while (__sync_lock_test_and_set(&lk->locked, 1) != 0)
        ;
    __sync_synchronize();
    lk->cpu = mycpu();
}
```

`acquire` 实现中有 `__sync_sychronize();` 这么一条语句，它保证了 `__sync_synchronize` 之前的指令不会在编译器优化时，被重排到 `__sync_synchronize` 之后，即避免了 critaical section 中的指令被重排到 `acquire` 之前。

在 `release` 的实现中同样有 `__sync_synchronzie` 语句，它又被称作 **memory barrier**，它基于硬件上的 **memroy fence** 实现。

## Using locks 

什么时候需要使用锁？一般来说，当多个 CPU（线程）需要访问同一块内存，并且 CPU（线程）对内存的操作涉及写入（而不仅仅是读取）时，就需要使用锁来保护这块内存上存储的数据，防止 race condition。

不涉及访问同一块内存，就不需要使用锁了吗？答案也是否定的，考虑在 `printf` 时，我们往往希望一个字符串能够被完整地输出而不与其他进程的 `printf` 交织。

## Deadlock and lock ordering

死锁个人认为就是指锁一直被持有，无法被释放。

最简单的死锁场景就是，一个进程重复申请持有同一个锁，例如：

```c
acquire(lk);
acquire(lk);
++i;
release(lk);
```

第二个 `acquire` 必须等第一个 `acquire` 状态被 `release` 了才能执行，但是不继续执行又无法走到第一个 `release`，就产生了死锁。xv6 会检查这种简单的死锁，触发 panic。

再考虑另一种死锁的情况，相比上一种情况更隐蔽，更难探测出来：

```c
// 进程 1
acquire(lk1);
acquire(lk2);
++i;
release(lk2);
release(lk1);

// 进程 2
acquire(lk2);
acquire(lk1);
++i;
release(lk1);
release(lk2);
```

那么就可能出现这样一种情况，proc1 持有了 `lk1`，然后 proc2 马上持有了 `lk2`，接着 proc1 申请持有 `lk2`，但是 `lk2` 已经被 proc2 持有了，只能等待 proc2 `release(lk2)`，然后 proc2 申请持有 `lk1`，同样的， `lk1` 被 proc1 持有了，因此必须等待 `proc1` `release(lk1)`，然而，两个进程都卡在第二个 `acquire` 这一步，无法执行 `release`，因此两个进程就只能一直等到地老天荒，这就发生了死锁。

为了避免出现这一问题，我们需要确定不同进程之间，锁对象有一个相同的申请顺序，即都先申请 `lk1`，再申请 `lk2`。

> If a code path through the kernel must hold several locks at the same time, it is important that all code paths acquire those locks in the same order. 

## Re-entrant locks

re-entrant locks 允许锁被**同一个进程**重复持有，又称为 **recursive locks**，使用 re-entrant locks 可以一定程度上避免死锁，然而，一般不推荐使用 re-entrant locks，因为这会让 debug 变得非常困难，产生意料之外的问题。

## Locks and interrupt handlers

spinlock 与 interrupt 的相互作用可能是非常危险的。假设 `sys_sleep` 持有了 `tickslock`，然后 cpu 被计时器中断打断，`clockintr` 会尝试获取 `tickslock`，然而由于 `tickslock` 已经被 `sys_sleep` 持有，那么 `clockintr` 会一直等待 `sys_spleep` 释放锁，然而在 `clockintr` return 之前，`sys_sleep` 是不会继续执行的，这就导致了死锁。

为了避免这一情况，如果 interrupt handler 需要使用 spinlock，那么当中断 enable 的时候，该 spinlock 绝不能被 cpu 持有。而 xv6更保守，在申请持有锁的时候，会 disable interrupt on that CPU。（注意这里只关闭同一个 cpu 的中断）

## Sleep locks

sleep locks 一般适用于线程需要长期持有锁的情况。例如 file system 需要向磁盘中写入文件，往往需要十多毫秒，如果使用 spin lock，意味着其他请求 spin lock 的线程会自旋等待十多毫秒且不会让出 cpu。

spin lock 的另一个缺点是持有 spin lock 的线程无法让出 cpu。因为 `acquire` 的时候会关闭 cpu 中断。

因此我们需要一种在线程 `acquire` 这个锁的时候，会让出 cpu 的锁，并且在持有锁的时候允许让出 cpu 以及中断响应。xv6 提供了这样一种锁，被称为 **sleep lock**。sleep lock 的具体实现细节见 lec11 & lec13。

