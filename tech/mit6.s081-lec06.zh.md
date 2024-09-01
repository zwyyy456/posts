---
title: "MIT 6.S081 Isolation & System call entry/exit"
date: 2023-07-08T15:32:43+08:00
lastmod: 2023-07-08T15:32:43+08:00 #更新时间
authors: ["zwyyy456"] #作者
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
## Trap 机制

程序运行往往需要完成用户空间和内核空间的切换，每当：

- 程序执行系统调用（system call）；
- 程序出现了 page fault 等错误；
- 一个设备触发了中断；

都会发生这样的切换。

这里用户空间切换到内核空间通常被称为 trap，因此有时候我们会说程序“陷入”到内核态。trap 机制需要尽可能的简单。

trap 的工作，可以说是让硬件从适合运行用户程序的状态，切换到适合运行内核代码的状态。

这里说的**状态**中，我们最关心的状态可能是 $32$ 个用户寄存器，我们尤其需要关注以下硬件寄存器的内容：

- 堆栈寄存器（stack register，又称 stack pointer）；
- 程序计数器（Program Counter Register）；
- 表明当前 mode 的标志位的寄存器，表明当前是 supervisor mode 还是 user mode；
- 控制 CPU 工作方式的寄存器，例如 SATP（Supervisor Address Translation and Protection）寄存器，它包含了指向 page table 的物理内存地址；
- STVEC（Supervisor Trap Vector Base Address Register）寄存器，它指向了内核中处理 trap 的指令的起始地址；
- SEPC（Supervisor Exception Program Counter）寄存器，在 trap 的过程中保存程序计数器的值；
- sscratch（Supervisor Scratch Register）寄存器；

在 trap 的最开始，CPU 所有的状态肯定还是在运行用户代码而不是内核代码，在 trap 处理的过程中，我们会逐渐更改状态，或者对状态做一些操作，我们可以设想一下我们需要做哪些操作：

- 保存 $32$ 个用户寄存器的状态，例如，当响应中断完成后，我们会希望能恢复用户程序的执行，而这些寄存器需要被内核代码所使用，因此，在 trap 之前，我们需要保存这 $32$ 个用户寄存器的内容；
- 保存 PC 的内容，原因类似于保存 $32$ 个用户寄存器；
- 将 mode 修改为 supervisor mode；
- 运行内核代码前，将 SATP 由指向 user page table 修改为指向 kernel page table；

trap 机制不会依赖于 $32$ 个用户寄存器；

supervisor mode 可以实现什么 user mode 不能实现的事情？（其实不多）

- 读写 SATP、STVEC、SEPC、sscratch 等寄存器；
- 使用 PTE_U 标志位为 0 的 PTE；

supervisor mode 并不能读写任意物理地址，在 supervisor mode 中，也需要通过 page table 来访问内存，如果一个物理地址映射的虚拟地址并不在当前 SATP 指向的 page table 中，又或者 SATP 指向的 page table 中，PTE_U = 1，那么 supervisor mode 不能使用那个地址。


## `ecall` 指令

2020 版的课程以 Xv6 的 `sh.c` 的 `getcmd` 中执行的 `write` 系统调用来说明这个例子（2021 版中，`getcmd` 转而使用了 `fprintf`），我这里其实是调试的 `echo.c`。

执行 `make CPUS=1 qemu-gdb` 以及 `riscv64-unknown-elf-gdb`（注意要配置好 `.gdbinit`），然后 gdb 中执行 `file user/echo.o` 以及 `b main`，将断点打在 `user/echo.c` 的 `main` 函数处，多执行几次 `continue`，直到 qemu 中的 shell 加载完成，可以执行命令了，输入 `echo zwyyy`，再在 `gdb` 的窗口中执行 `layout split` 以及 `continue`，函数将停在 `main` 函数处，从 `echo.asm` 中我们可以看到 `write` 系统调用对应的 ECALL 指令所在的地址，为 $\text{0x31c}$，因此我们执行 `b *0x31c`，然后执行 `continue`：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065846.png)

然后我们打印 PC 的值，正好在 $\text{0x31c}$ 处：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065849.png)

a0，a1，a2 寄存器中的内容是 shell 传递给 `write` 系统调用的参数，a0 是文件描述符，a1 是 shell 想要写入的字符串的指针，a2 是想要写入的字符数：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065851.png)

在 QEMU 中输入 `ctrl + a`，再输入 `c` 可以进入到 QEMU 的 console 中，之后输入 `info mem`，QEMU 会打印完整的 page table

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065852.png)

可以看到最后两个 pte 的 vaddr 非常大，接近虚拟地址的顶端，这两个 page 分别是 trapframe page 和 trampoline page。

然后我们输入 `stepi` 来执行 `ecall` 指令，此时输入`print $pc`，发现 pc 中的内容是一个很大的地址，它正是 trampoline page 的最开始，我们的指令正在内存的 trampline page 中。

`ecall` 指令的下一个指令是 `csrrw a0, sscratch, a0`，交换了寄存器 a0 和寄存器 sscratch 的内容。

我们现在正在 trampoline page 中，ecall 并不会切换 page table，因此，trap 处理代码必须存在于每个 user page table 中，因为 ecall 并不会切换 page table，因此，我们需要在 user page table 中的某处来执行最初的内核代码，而这个 trampoline page 由内核映射到每一个 user page table 中，使得我们仍在使用 user page table 时，内核能在一个地方执行 trap 机制的最开始的一些指令。

这里的指令从 `ecall` 跳转到 trampoline page 是通过 `ecall` 指令实现的：

ECALL 指令其实只会做三件事：

- 将代码从 user mode 切换到 supervisor mode；
- 将 PC 的值保存在 SEPC 寄存器中；
- 跳转到 STVEC 寄存器指向的指令，STVEC 寄存器只能在 supervisor mode 下进行读写；

打印 sepc 寄存器的内容，我们可以看到这个寄存器的内容就是 `ecall` 指令在用户空间的地址。

![jvAcpDMgtbnmGTq](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065854.png)

## `ecall` 指令执行完之后

`ecall` 帮我们做了开头的工作，但离我们执行内核中的 C 代码还有很多工作要做：

- 我们需要保存 $32$ 个用户寄存器的内容，这样当我们需要恢复执行用户代码时，我们才能恢复这些寄存器的内容，从而让用户代码察觉到不到自己被暂停执行系统调用去了；
- 从 user page table 切换到 kernel page table
- 创建或者找到一个 kernel stack，将 stack pointer 寄存器的内容指向那个 kernel stack，这样内核中的 C 代码才有栈可以使用；
- 还需要跳转到内核中 C 代码某些合理的位置；

`ecall` 指令之所以不完成上述的工作，是因为 RISC-V 的设计理念，即“`ecall` 只尽量完成少的必须要完成的工作，其他工作都交给软件完成，从而为软件与操作系统程序员提供最大的灵活性”。

## `uservec` 函数 与 trampoline page

Xv6 为每个 user page table 映射了 trapframe page，在 trapfram page 中有 $32$ 个空槽位，用来保存 $32$ 个用户寄存器的内容。

trapframe page 在 user page table 中的虚拟地址总是 $\text{0x3ffffffe000}$。

如果想要查看 Xv6 在 trapframe page 中存放了什么，可以查看 `proc.h` 中的 `trapframe` 结构体。

![FVieGEDny1NXvhp](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065857.png)

保存用户寄存器的方案分为两部分： 

- 内核将 trapframe page 映射到了每个 user page table；
- 另一部分则在于 sscratch 寄存器；

在进入 user space 之前，内核会将 trapframe page 的地址保存在这个寄存器中，即 $\text{0x3ffffffe000}$。

注意，我们只会**在 user page table 下使用 trapframe page 的虚拟地址**！后面可以看到，在 `userret` 时，我们是将 kernel page table 切换回 `user page table` 之后才使用的 `trapframe page`！

我们可以看到，当执行完 `csrrw` 指令之后，a0 寄存器中的值就是 trapframe page 的虚拟地址，此后的一系列 `sd` 指令，就是将寄存器的内容保存在 trapframe page 的不同 offset 处，即 $offset + a0$ 处。

![4a6pAfmx157UTDk](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065858.png)

在内核上一次切换回用户空间时，sscratch 寄存器的内容会被设置为 $\text{0x3ffffffe000}$。

Xv6 的启动流程参见 [MIT 6.S081 操作系统组织架构](https://blog.zwyyy456.tech/zh/posts/tech/xv6-os-organization/)，会经过一个从 machine mode 到 supervisor mode 再到 user mode 的过程，因此在 `ecall` 指令执行之前，sscratch 的内容就已经被设置好了。

在第一个 `ld` 指令之前，有 `csrr t0, sscratch` 和 `sd t0, 112(a0)` 这两条指令。这两条指令结合起来，就是将 sscratch 的值写入到了 `trapframe` 结构体中的 `a0` 字段处。

然后我们将停在第一个 `ld` 指令处，这条指令将 a0 中的内存地址（即 trapframe 的地址）往后数 $8$ 个字节处开始的数据加载到 Stack Pointer 寄存器中，由 `trapframe` 结构体我们可以知道，这里的数据就是内核的 Stack Pointer 的内容。

![j3MWkbvOis7FDeX](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065900.png)

trapframe 中的 kernel_sp 是由 kernel 在进入用户空间之前就设置好的，它的值是这个进程的 kernel stack。所以这条指令的作用是初始化 Stack Pointer 指向这个进程的 kernel stack 的最顶端。指向完这条指令之后，我们打印一下当前的 Stack Pointer 寄存器。

下一条指令是将 `trapframe` 结构体中的 `kernel_hartid` 写到 tp 寄存器中，`kernel_hartid` 就是当前 CPU 核的编号。

再下一条指令是往 t0 寄存器中写入数据，这里写入的是将要执行的第一个 C 函数 —— `usertrap` 的指针。

再下一条指令是向 t1 中写入包含 kernel page table 地址的数据（相比真正的地址，进行了移位）。

再下一条交换 SATP 寄存器和 t1 寄存器，当这条指令完成后，当前程序会从 user page table 切换到 kernel page table，而 t1 寄存器中保存着 user page table 的地址。

我们已经切换了 page table，而程序计数器中保存的是虚拟地址，但我们还是能正确执行内容，同一个虚拟地址不会因为 page table 的切换而走到无关的 pp 中，这正是因为我们还在 trampoline page 中，而 **trampoline page 在用户空间和内核空间都映射到了同一个虚拟地址**。

**只有 trampoline page 在 user page table 和 kernel page table 中都映射到了同一个虚拟地址！**

最后一条指令是 `jr t0`，执行了这条指令，我们就从 trampoline page 跳转到内核的 C 代码中去了。

## `usertrap` 函数

`usertrap` 函数定义于 `kernel/trap.c` 中：

![5LNSeEWnpHQGP16](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065902.png)

首先，我们执行 `w_stvec`，从而将`kernelvec` 变量的值写入到  STVEC 中，这是内核空间 trap 处理代码的位置，而不是用户空间 trap 处理代码的位置，从而保证 trap 来源于内核空间时，不会像来源于用户空间时那样执行很多不必要的操作。

然后，我们通过 `myproc` 函数获取当前进程编号，接下来我们将 SEPC 寄存器中的值保存到 `p->trapframe->epc` 中，防止中途切换到另一个要执行系统调用的进程中导致 SEPC 被覆盖。

接下来我们检查 SCAUSE 寄存器，如果内容为 $8$，说明是通过系统调用触发 trap 的，然后检查是否有其他进程杀掉了当前进程。

我们将 `p->trapframe->epc += 4`，是因为我们希望恢复到用户代码时，不是恢复到 `ecall` 然后再执行一遍 `ecall`，而是用户代码中 `ecall` 的下一条指令，即 `ret`。

XV6 会在处理系统调用的时候使能中断，这样中断可以更快的服务，有些系统调用需要许多时间处理。中断总是会被 RISC-V 的 trap 硬件关闭，所以在这个时间点，我们需要执行 `intr_on()` 显式的打开中断。

之后，我们会调用 `syscall` 函数，`syscall` 从 a7 寄存器中获取系统调用编号，而之前系统调用 `write` 对应的汇编代码中，有一个 ` li a7, SYS_write`，因此 `syscall` 就知道要去执行 `sys_write()` 函数了（`syscall.c` 中定义了一个索引表）。

`write` 的参数在 `ecall` 之前分别保存在 a0、a1、a2 处，在 trap 机制之后，它们分别位于 `trapframe` 结构体的 a0、a1、a2 字段处。

`p->trapframe->a0 = syscalls[num]();` 这里向 `trapframe` 中的 a0 赋值是因为：所有系统调用都会返回一个返回值，RISC-V 上的 C 代码习惯将函数返回值存储于寄存器 a0，而返回到用户空间时，trapframe 的 a0 槽位的数值会写回到实际的 a0 寄存器，shell 会认为 a0 寄存器的值就是 `write` 系统调用的返回值。

在 `usertrap` 函数中，执行完 `syscall` 之后，我们再次检查进程是否被杀掉（我们不希望恢复一个被杀掉的进程），然后执行 `usertrapret`。

## `usertrapret` 函数

首先，它执行 `intr_off` 关闭了中断。

然后，执行 `w_stvec(TRAMPOLINE + (uservec - trampoline));` 将 STVEC 寄存器中的内容修改为指向 user trampoline 的地址。

然后执行一系列操作将对应值填入到 `trapframe` 中，再之后设置 SSTATUS 寄存器，该寄存器的 SSP bit 为 $0$ 说明我们执行 `sret` 时应该返回 user mode 而不是 supervisor mode。SPIE bit 为 $1$ 表明执行 `sret` 之后是否打开中断。

然后我们将 `p->trapframe->epc` 的值写入到 SEPC 寄存器中去。

`uint64 fn = TRAMPOLINE + (userret - trampoline)` 计算出我们要跳转到的汇编代码的地址，即 trampoline page 中的 `userret` 函数。

再将相应 user page table 地址写入到 `satp`，然后通过 `fn` 函数传递，**`satp` 的值会出现在 a1 寄存器中**，执行该 `userret` 函数。

![uer61MBGQ8dgtlz](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065904.png)

## `userret` 函数

`userret` 函数位于 `trampoline.S` 中

![Z5XIy2YRt1ck43i](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065905.png)

第一步是将 page table 从 kernel page table 切换到 user page table 中，即**将 a1 寄存器的值写入到 satp 寄存器中去**，**a1 寄存器的值就是 user page table 的地址**。

然后我们执行 `ld t0, 112(a0)` 和 `csrw sscratch, t0`，将 `p->trapframe->a0` 的值写入到了 sscratch 寄存器中。

之后是将 `trapframe` 中的一系列值写回到寄存器中去。

![KBL6lcbOSnfaq9A](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065908.png)

此时 a0 寄存器仍然是 TRAPFRAME 的地址，而 sscratch 寄存器中则是系统调用的返回值，执行一次 `csrrw a0, sscratch, a0`，交换两个寄存器的值，sscratch 中就存储着 TRAPFRAME 的地址了，而 a0 寄存器中则是系统调用的返回值。

`sret` 是我们在 kernel 中的最后一条指令，执行它会产生以下三个效果：

- 程序切换回 user mode；
- SEPC 寄存器的数值被保存到 PC 寄存器，它指向 `ecall` 的下一条指令 `ret`；
- 重新打开中断；
