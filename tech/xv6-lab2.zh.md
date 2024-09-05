---
title: "Xv6 Lab2: System calls"
date: 2023-07-01T15:15:38+08:00
lastmod: 2023-07-01T15:15:38+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
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
## 系统调用

Lab1 主要是基于提供的系统调用接口来编写一些小工具程序，而 Lab2 则是要我们自己实现系统调用，并提供系统调用的接口。

以本次 Lab 要我们实现的 `trace` 调用为例，说明一下系统调用的流程：

在 `user/trace.c` 的第 $15$ 行，调用了属于 system call 的 `trace` 函数，当前执行 `make qemu` 是无法成功的，因为我们还没有给用户提供接口。因此，我们需要在 `user/user.h` 里面添加系统调用 `trace` 的函数声明（prototype）。

我们需要在 `user/usys.pl` 中追加 `entry("trace");` 这一行，从而添加一个 **trap entry**，从而实现调用 `trace` 时会发生 **trap**，从而会执行 `ECALL` 指令，并且会将系统调用的接口的参数（这里就是 `trace` 的参数）的存入寄存器 `a0`、`a1` 等（从左往右第一个参数的地址存入 `a0`，依次类推），此外，还会将 `trace` 对应的系统调用号存入寄存器 `a7`。

之后控制权来到 kernel 中的 `syscall` 函数，它从 `a7` 中取出系统调用号，并执行系统调用号对应的 `sys_func`。

> 这里的系统调用号可以理解为数组索引，在本次 Lab 中需要我们修改 `kernel/syscall.c` 和 `kenel/syscall.h` 从而添加 `trace` 对应的系统调用号，以及内核中 `trace` 对应的 sys_trace 的实现）。

## System call tracing

官网的提示其实是比较全面了，按提示处理，即可添加 `trace` 的系统调用接口：

> Run make qemu and you will see that the compiler cannot compile user/trace.c, because the user-space stubs for the system call don't exist yet: add a prototype for the system call to user/user.h, a stub to user/usys.pl, and a syscall number to kernel/syscall.h. The Makefile invokes the perl script user/usys.pl, which produces user/usys.S, the actual system call stubs, which use the RISC-V ecall instruction to transition to the kernel. Once you fix the compilation issues, run trace 32 grep hello README; it will fail because you haven't implemented the system call in the kernel yet. 

之后，则需要我们自己在 `kernel/syspro.c` 中实现真正执行系统调用 `trace` 所需要执行的操作了：

Lab 给出的提示中提到，我们需要将 `trace(int mask)` 的参数 `mask` 参数存放到 `proc` 结构体中的一个新成员变量中，这里我们直接用 `mask` 表示，于是，`sys_trace` 要做的其实很简单，就是取出寄存器 `a0` 中的地址然后解引用，得到 `mask`，然后将它赋给 `proc` 结构体中的 `mask`。

对于取出寄存器 `a0` 的参数，Xv6 已经给我们封装好了一个函数 `argint`，参照一下 `sys_exit` 的实现，以及 `int argint(int n, int *p)` 的实现，就能知道 `argint` 如何使用了，第一个参数表示从哪个寄存器中取值，取出来的值放在地址 `p` 中。

```c
// 修改 syscall.h，添加 #define SYS_trace 22
static uint64 (*syscalls[])(void) = {
    [SYS_fork] sys_fork,
    [SYS_exit] sys_exit,
    [SYS_wait] sys_wait,
    [SYS_pipe] sys_pipe,
    [SYS_read] sys_read,
    [SYS_kill] sys_kill,
    [SYS_exec] sys_exec,
    [SYS_fstat] sys_fstat,
    [SYS_chdir] sys_chdir,
    [SYS_dup] sys_dup,
    [SYS_getpid] sys_getpid,
    [SYS_sbrk] sys_sbrk,
    [SYS_sleep] sys_sleep,
    [SYS_uptime] sys_uptime,
    [SYS_open] sys_open,
    [SYS_write] sys_write,
    [SYS_mknod] sys_mknod,
    [SYS_unlink] sys_unlink,
    [SYS_link] sys_link,
    [SYS_mkdir] sys_mkdir,
    [SYS_close] sys_close,
    [SYS_trace] sys_trace,
};

// syscall for trace
uint64 sys_trace(void) {
    struct proc *to_trace = myproc();
    int mask;
    if (argint(0, &mask) < 0) { // 取出 trace 之后的 mask 参数
        return -1;
    }
    to_trace->mask = mask;
    return 0;
}
```

然后需要修改 `fork()`，从而使得子进程也共享同一个 `mask`，这里很简单，在 `fork()` 的实现中添加 `np->mask = p->mask;` 即可。

最后还需要修改 `syscall()` 函数，从而当系统调用号属于 `mask` 使能的系统调用号时，按 Lab 要求进行 `printf`。

```c
void syscall(void) {
    int num;
    struct proc *p = myproc();

    num = p->trapframe->a7;
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        uint64 res = syscalls[num]();
        p->trapframe->a0 = res;
        int mask = p->mask;
        if ((1 << num) & mask) {
            printf("%d: syscall %s -> %d\n", p->pid, name[num], res);
        }
    } else {
        printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}
```

## Sysinfo

添加接口的操作同上，我们先实现 `sys_sysinfo`，由于 `sysinfo` 接口的参数是个指针（或者说地址），尽管该参数的地址依旧存放于 `a0` 寄存器中，但是我们不能再使用 `argint` 了，这里 Lab 给出了提示，参照 `sys_fstat` 的实现，我们知道了可以使用 `argaddr` 实现这一操作，`a0` 中的值被存放于 `uint64 addr` 中。

之后，我们需要创建一个新的 `struct sysinfo` 结构体变量 `info`，然后将统计的 amount of free memory 和 number of processes 存放于这个结构体变量中。

之后，Lab 提示我们使用 `copyout` 将 `info` 传递到 user space，其实参照 `copyout` 的注释就能清楚明白 `copyout` 是干嘛的，怎么用了。

```c
uint64 sys_sysinfo(void) {
    uint64 addr; // user pointer to struct sysinfo
    if (argaddr(0, &addr) < 0) {
        return -1;
    }
    struct proc *p = myproc();
    struct sysinfo info;
    info.nproc = get_proc_num();
    info.freemem = freemem_size();
    if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0) {
        return -1;
    }
    return 0;
}
```

在 `defs.h` 中添加 `get_proc_num` 和 `freemem_size` 的 prototype。

分别在 `kernel/kalloc.c` 和 `kernel/proc.c` 中实现。统计 amount of free memory，从 `kalloc.c` 中可以发现，Xv6 是使用 external free memory list 的形式来进行 `malloc` 的，所以我们只需要统计 free memory list 的结点数就能知道了；而对统计 number of processes 来说，从 `proc.c` 的 `struct proc proc[NPROC];` 中可以得知，进程是以数组形式管理的，我们统计该数组中状态不为 `unused` 即可知 number of processes。

```c
// kernel/kalloc.c
int freemem_size(void) {
    struct run *r;
    int num = 0;
    for (r = kmem.freelist; r; r = r->next) {
        ++num;
    }
    return num * PGSIZE;
}

// kernel/proc
int get_proc_num(void) {
    struct proc *p;
    int num = 0;
    for (p = proc; p < &proc[NPROC]; ++p) {
        if (p->state != UNUSED) {
            ++num;
        }
    }
    return num;
}
```

## 总结

这个 Lab 设计得非常精妙，做了 System call tracing 之后，对系统调用的流程就有了比较深刻的理解了，Sysinfo 则是加深理解，两个实验一起，对系统调用时，参数的传入、结果的传出，都有了大致概念。


