---
title: "Xv6 Lab4: Traps"
date: 2023-07-13T12:51:50+08:00
lastmod: 2023-07-13T12:51:50+08:00 #更新时间
authos: ["zwyyy456"] #作者
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
## RISC-V assembly

Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf? 

- a2 寄存器，函数调用时，参数从左到右会依次保存在 a0, a1, a2, a3 寄存器，似乎是一直到寄存器 a7 的。

Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.) 

- 这里的调用都被内联了

At what address is the function printf located? 

- `auipc`（Add Upper Immediate PC）指令是将一个立即数左移 $12$ 位加上当前指令的地址（pc）中，得到一个绝对地址。

- 例如 `auipc a0, 0x0` 就是将 $\text{0x0}$ 左移 $12$ 位加上当前指令的地址 (pc) 中（pc 的值我们一般认为是在指令执行完成时才发生递增，从而指向下一条指令，使得处理器能够按顺序顺利执行指令序列），因此，执行 `auipc a0, 0x0` 时，加的就是当前指令的地址，即 $\text{0x28}$。

- ![Ep4uySNWrac6nwQ](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065818.png)

- `auipc` 常常会与 `jalr`（Jump and Link Register）指令组合在一起，在 `jalr 1526(ra)` 中，它把 pc 设置为寄存器 ra 的值加上 offset，然后把原先的 $pc + 4$ 的值写入寄存器 ra，即 ra 的值就是函数的返回值，即正常序列里，`jalr` 的下一条指令，这里 `ra` 的值应该是 $\text{0x38}$, pc 的值则为 $\text{0x626}$。

```c
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

- 上面这段代码的输出结果为 `He110 World`。

## Backtrace 

这里的关键在于弄懂 stack frame 的结构，参考这张图，其实基本原理与 x86 下 C 语言的 stack frame 是一致的，我们通过 `uint64 fp = r_fp()` 函数拿到的是栈基地址（联想 x86 中的 rbp），因此 `fp + 8` 就是 return address 的地址，因此 `uint64 ra = *(uint64 *)(fp + 8);`，注意 `fp + 8` 是地址的地址，不是我们要输出的 ra！

我们需要遍历 stack frame，上一个 stack frame 的栈基地址就存放在 `fp + 16` 处，因此 `fp = *(uint64 *)(fp + 16);`。

循环的终止条件，可以先计算出 `fp` 的上下界 `PGROUNDDOWN(fp)` 和 `PGROUNDUP(fp)`（按照 PGSIZE 大小对齐），又或者当 `PGROUNDUP(fp) != p->kstack + PGSIZE` 时退出循环。

```cpp
void backtrace(void) {
    uint64 fp = r_fp();
    printf("backtrace\n");
    uint64 upb = PGROUNDUP(fp);
    uint64 db = PGROUNDDOWN(fp);
    // struct proc *p = myproc();
    while (fp < upb && fp > db) {
        uint64 ra = *(uint64 *)(fp - 8);
        printf("%p\n", ra);
        fp = *(uint64 *)(fp - 16);
    }
}
```

## Alarm

这个实验在了解了 trap 机制的整个流程，如何陷入到内核代码，又如何返回用户代码之后，其实并不难，但是还是有两个地方比较难想：

1. `sigalarm` 的调用其实只起到一个开关的作用，当开启 `sigalarm` 之后，还是要在 `usertrap` 的外部定时中断部分来执行定时响应函数，我们可以通过 `argaddr` 函数来获取 `sigalarm` 传入的作为函数地址的第二个参数，但是，我们并不能直接在 `usertrap` 中把函数地址强制类型转化为函数指针来调用函数，这是因为 `kernel page table` 与 `user page table` 是不一样的，我们需要把 `p->trapframe->epc` 修改为 `periodic` 的地址，在返回用户态时，pc 会被更新为 `p->trapframe->epc` 的值，这样返回到用户态时就会马上执行 `periodic` 函数；

2. 经过上面的操作，我们返回时会直接执行 `periodic` 函数，但是执行完函数之后，机器就不知道接下来该执行哪一条指令来，因此提供了 `sigreturn` 函数，使得可以恢复到发生定时器中断时的状态，这里提到了需要保存很多寄存器的值，因此直接在 `struct proc` 中添加一个 `struct trapframe *save_reg;` 字段，在 `usertrap` 的定时中断响应中，利用 `memmove` 将 `p->trapframe` 复制到 `p->save_reg`，而在 `sys_return` 中将 `p->save_reg` 恢复到 `p->trapframe`。

几处关键的代码：

`trap.c` 中：

```c
void usertrap(void) {
    int which_dev = 0;

    if ((r_sstatus() & SSTATUS_SPP) != 0)
        panic("usertrap: not from user mode");

    // send interrupts and exceptions to kerneltrap(),
    // since we're now in the kernel.
    w_stvec((uint64)kernelvec);

    struct proc *p = myproc();

    // save user program counter.
    p->trapframe->epc = r_sepc();

    if (r_scause() == 8) {
        // system call

        if (p->killed)
            exit(-1);

        // sepc points to the ecall instruction,
        // but we want to return to the next instruction.
        p->trapframe->epc += 4;

        // an interrupt will change sstatus &c registers,
        // so don't enable until done with those registers.
        intr_on();

        syscall();
    } else if ((which_dev = devintr()) != 0) {
        // ok
        if (which_dev == 2 && p->interval != 0) {
            // 说明启用了 alarm
            ++p->tick_num;
            if (p->tick_num == p->interval) {
                if (p->flag == 0) {
                    // typedef void (*Handler)();
                    // printf("p->handler: %p\n", p->handler);
                    memmove(p->save_reg, p->trapframe, PGSIZE);
                    p->flag = 1;
                    p->trapframe->epc = p->handler;
                }
                p->tick_num = 0;
            }
        }
    } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
        printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
        p->killed = 1;
    }

    if (p->killed)
        exit(-1);

    // give up the CPU if this is a timer interrupt.
    if (which_dev == 2) {
        yield();
    }
    usertrapret();
}
```

`sysproc.c` 中：

```c

uint64 sys_sigalarm() {
    struct proc *p = myproc();
    int interval;
    if (argint(0, &interval) < 0) {
        return -1;
    }
    uint64 addr;
    if (argaddr(1, &addr) < 0) {
        return -1;
    }
    p->interval = interval;
    p->handler = addr;

    return 0;
}

uint64 sys_sigreturn() {
    struct proc *p = myproc();
    memmove(p->trapframe, p->save_reg, PGSIZE);
    p->flag = 0;
    return 0;
}
```

同时注意在 `proc.c` 的 `allocproc` 中给 `save_reg` 分配空间，在 `freeproc` 中释放。

以及 `kernel/param.h` 中，`FSSIZE` 要修改为至少 $1088$，否则 `alarmtest` 能过，但是 `usertests` 过不了，会卡在 writebig 处，报错提示为 "test writebig: panic: balloc: out of blocks"，不知道有没有大佬能解惑。

另外，为什么一定要保存整个 `trapframe` 呢？









