---
title: "MIT 6.S081 操作系统组织架构"
date: 2023-06-29T18:59:04+08:00
lastmod: 2023-06-29T18:59:04+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["xv6","mit"]
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
## 进程概述

64 位的 RISC-V 的 VAS 是 39 位的，即 VA 只有 39 位，而 Xv6 则只有 38 位，最大虚拟地址为 `#define MAXVA 0x3fffffffff`。

VAS 的顶端，即最高位存放了两个 page，一个是用于 *trampoline*，一个用于 mapping the process's trapframe。 Xv6 使用这两个 page 来切换到内核以及返回。

进程的状态被定义在 `kernel/proc.h` 的结构体 `struct proc` 所描述，进程在内核中最重要的信息就是它的 page table、 kernel stack 以及运行状态（run state）。

A process can make a system call by executing the RISC-V ecall instruction. This instruction
raises the hardware privilege level and changes the **program counter** to a **kernel-defined entry point.**

The code at the entry point **switches to a kernel stack** and executes the kernel instructions that
implement the system call. When the system call completes, the kernel switches back to the user
stack and returns to user space by calling the `sret` instruction. (x64 架构似乎是 `iret`？)

![tNX75y3sYmVHobl](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065751.png)

## User/Kernel mode 切换

我们是如何从 user mode 进入到 kernel mode 的？

例如，当 `ls` 程序运行时，会调用 `read`/`write` 系统调用；shell 程序会调用 `fork` 或者 `exec` 系统调用，所以必须要有一张方式可以使得用户的应用程序能够将控制权以一种协同工作的方式转移到内核，这样内核才能提供相应的服务。

在 RISC-V 中，有一个专门的指令用于实现这一功能，叫做 `ECALL`（指令而非函数？），`ECALL` 接收一个数字参数，当一个用户程序想要将程序的执行控制权转移到内核，它只需要执行 `ECALL` 指令，并且传入一个数字。这里的数字参数代表了应用程序想要调用的 `System Call`。 `System Call` 表如下所示：

```c
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
```

`ECALL` 会跳转到内核中一个特定的、由内核控制的位置，在 Xv6 中存在一个唯一的系统调用接入点，每一次应用程序执行 `ECALL` 指令，应用程序都会通过这个接入点进入到内核中。例如，不论是 shell 还是其他的应用程序，当它在用户空间执行 `fork` 时，它并不是直接调用操作系统中对应的函数，而是调用 `ECALL` 指令，并将 `fork` 对应的数字 `SYS_fork` 作为参数传给 `ECALL`，之后再通过 `ECALL` 跳转到内核中。

在内核侧，有一个位于 `syscall.c` 的函数 `syscall`，每一个从用户程序发起的系统调用都会调用到这个 `syscall` 函数，`syscall` 函数会检查 `ECALL` 的参数，通过这个参数，内核可以知道要调用的是 `fork`。

假设现在要执行另外一个系统调用 `exec`，两个参数，用户代码会将 `exec` 需要的参数存放在寄存器 `a0` 和 `a1` 中，事实上这里的两个参数都是地址，并将系统调用号存放在寄存器 `a7` 中。

用户态中执行的 `exec` 系统调用并不能直接执行内核中的 `exec` 代码，而是要由封装好的系统调用函数执行 `ECALL` 指令，所以 `exec` 函数实际上调用的是 `ECALL` 指令，指令的参数代表了 `exec` 系统调用的索引数字，之后控制权到了 `syscall` 函数，`syscall` 函数才实际调用 `exec` 系统调用。

## Xv6 启动流程

RISC-V 启动时，先运行一个存储于 ROM 中的 bootloader 程序 `kernel.ld` 来加载 `xv6 kernel` 到内存中，然后在 `machine` 模式下从 `_entry` 开始运行 Xv6。bootloader 将 xv6 kernel 加载到 0x80000000 的物理地址中，因为前面的地址中有 I/O 设备

在 `_entry` 中设置了一个初始 stack，`stack0` 来让 Xv6 执行 `kernel/start.c`。在 `start` 函数先在 machine 模式下做一些配置，然后通过 RISC-V 提供的 `mret` 指令切换到 supervisor mode，使 program counter 切换到 `kernel/main.c`。

`main` 先对一些设备和子系统进行初始化，然后调用 `kernel/proc.c` 中定义的 `userinit` 来创建第一个用户进程。这个进程执行了一个 `initcode.S` 的汇编程序，这个汇编程序调用了 `exec` 这个 `system call` 来执行 `/init`，重新进入 kernel。`exec` 将当前进程的内存和寄存器替换为一个新的程序（`/init`），当kernel执行完毕exec指定的程序后，回到 `/init` 进程。`/init`（`user/init.c`）创建了一个新的 console device 以文件描述符 0, 1, 2 打开，然后在 console device 中开启了一个 shell 进程，至此整个系统启动了。

