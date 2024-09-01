---
title: "Xv6 Lab5: lazy page allocation"
date: 2023-07-15T17:18:30+08:00
lastmod: 2023-07-15T17:18:30+08:00 #更新时间
authors: ["zwyyy456"] #作者
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
## 前言

这个实验只有 2020 年的才有，2021 年的课程中是没有的，但是感觉这个实验还是挺有意义的，因此用 docker 创建了一个 debian 12 的容器，在容器中搭建了 2020 的实验环境，实验环境的搭建过程可以参照 [MIT 6.s081 实验环境搭建](https://blog.zwyyy456.tech/zh/posts/tech/6-s081-env-configuration/)。

## Eliminate allocation from sbrk()

这个算是最简单的：

```c
// kernel/sysproc.c
uint64 sys_sbrk(void) {
    int addr;
    int n;
    if (argint(0, &n) < 0) {
        return -1;
    }
    addr = myproc->sz;
    myproc()->sz += n; // 添加的部分，修改 p->sz，然后注释掉下面这三行
    // if (growproc(n) < 0) {
    //     return -1;
    // }
    return addr;
}
```

## Lazy allocation

在去掉了 `sys_sbrk` 的 `growproc` 部分之后，由于只是单纯增加了 `p->sz`，而没有给对应的虚拟地址分配物理页，因此，在执行 `echo hi` 时，会访问到 heap 中的未分配物理页的虚拟地址，于是出现 page fault，默认的 Xv6 的代码中并没有给出对 page fault 的处理，而是会直接杀死进程，因此无法正常执行完 `echo hi`。

因此，我们需要修改 `kernel/trap.c` 中的 `usertrap` 函数，为它添加对 page fault 的处理函数，提示已经说的很清楚了，利用 `r_scause()` 函数读取 scause 寄存器，如果值为 $13$ 或者 $15$ 就说明是 page fault，然后执行相应的处理函数。

我们可以观察一下 `growproc` 在 `n > 0` 的时候做了什么：调用 `uvmalloc` 并更新 `p->sz` 为 `sz + n`，而 `uvmalloc` 函数其实就是调用 `mappages` 来为虚拟地址对应的 VP 分配对应的 PP。

因此，在 page fault 的处理部分，我们可以参照 `uvmalloc` 给虚拟地址对应的 VP 分配对应的 PP，同时要注意检查虚拟地址是不是一个有效的虚拟地址。

```c
// else if ((which_dev = devintr()) != 0) {
    // ok
// }
else if (r_scause() == 15 || r_scause() == 13) {
        uint64 wrong_addr = r_stval();
        if (wrong_addr >= p->sz) { // va is not allocated by sbrk()
            p->killed = 1;
            exit(-1);
        }
        if (wrong_addr < PGROUNDUP(p->trapframe->sp)) { // wrong_addr 必须在 heap，不能在 stack 中
            printf("under stack!\n");
            p->killed = 1;
            exit(-1);
        }
        uint64 lb = PGROUNDDOWN(wrong_addr);
        char *mem = kalloc();
        if (mem == 0) {
            // printf("there is not enough free space\n");
            p->killed = 1;
        } else {
            memset(mem, 0, PGSIZE);
            // printf("memset\n");
            if (mappages(p->pagetable, lb, PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
                kfree(mem);
                uvmdealloc(p->pagetable, lb + PGSIZE, lb);
                p->killed = 1;
            }
        }
    } 
```

此外，我们还需要修改 `uvmunmap`，否则会出现 panic：

Xv6 中，由于 `sbrk` 是以 eager 的形式扩充 heap 的，所有的虚拟地址都被映射到了物理地址上，因此，不应该存在 valid 标志位不为 $1$ 的情况，因此若出现该情况，直接 `panic`；

但是，当我们采取 lazy 的方式进行内存分配时，存在虚拟地址没有被物理地址映射是正常情况，因此 `(pte = walk(pagetable, a, 0)) == 0` 和 `(*pte & PTE_V) == 0` 都是正常现象：
- 前者可能虚拟地址对应的 L1, L0 页表都还不存在，又或者只有 L1 页表存在，L0 页表还未分配；
- 后者表示 L1, L0 页表都存在，但是虚拟地址还没有被物理地址映射，即 L0 页表的其他的 pte 对应的虚拟地址可能已经被物理地址映射了。

因此，碰到这两种情况，我们直接 `continue;` 即可：

```c
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free) {
    uint64 a;
    pte_t *pte;

    if ((va % PGSIZE) != 0)
        panic("uvmunmap: not aligned");
    for (a = va; a < va + npages * PGSIZE; a += PGSIZE) {
        if ((pte = walk(pagetable, a, 0)) == 0) { // 说明 pte 是无效的
            continue;                             // 未分配的页无需释放
            panic("uvmunmap: walk");
        }
        if ((*pte & PTE_V) == 0)
            // panic("uvmunmap: not mapped");
            // printf("may panic\n");
            // 对未分配的页跳过
            continue;

        if (PTE_FLAGS(*pte) == PTE_V)
            panic("uvmunmap: not a leaf");
        if (do_free) {
            uint64 pa = PTE2PA(*pte);
            kfree((void *)pa);
        }
        *pte = 0;
    }
}
```

## Layztests and Usertests

这个实验还是有些难度的，关键是有个地方要能想到，我们先跟着实验的提示来：

> 处理 `sbrk` 的参数为负数的情况: 

这个很好处理，负数的时候，调用 `growproc` 即可：

```c
uint64 sys_sbrk(void) {
    int addr;
    int n;

    if (argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    if (n < 0) {
        if (growproc(n) < 0) {
            return -1;
        }
        return addr;
    }
    myproc()->sz += n;
    return addr;
}
```

> 处理访问的虚拟地址没有被 `sbrk` 分配的情况：

这里其实是有一个要注意的细节的，我们的虚拟内存一般是从 $[0, 2^{n} - 1]$，或者说 $[0, 2^{n})$，因此，可以说虚拟内存空间一般是**左闭右开**的，而 heap，stack 等，假设它们都占据一个 `PGSIZE` 的大小，那么，可以说它们从低地址到高地址也是左闭右开的。因此，我们执行 `sbrk` 更新得到的地址 `p->sz` 应该是不能被访问的，相当于是开区间的右端点（当然，由于我们一次至少分配 `PGSIZE` 的大小，除非 `p->sz` 也是 `PGSIZE` 对齐的，不然实际上还是可以访问的）

>处理虚拟地址位于 stack 的情况：

这里一开始我不知道怎么找到 stack 的虚拟地址，后面我查看了一下 `usertests.c` 的 `stacktest` 函数，里面是通过 `r_sp()` 获取 sp 寄存器的地址，来获取 stack 的栈顶的虚拟地址，而在发生 trap 时，sp 寄存器的值会被保存到 `p->trapframe->sp` 中，因此，在内核中我们可以通过读取 `p->trapframe->sp` 来获取当前栈顶指针的值（即栈顶的地址），我们再使用 `PGROUNDUP` 进行对齐，注意之前提到的，stack 也是低地址到高地址左闭右开，因此 **`PGROUNDUP(p->trapframe->sp)` 这个地址是属于 heap 的**。

因此，我们的 `usertrap` 函数的 page fault 的处理部分应该是这样的：

```c
} else if (r_scause() == 15 || r_scause() == 13) {
    uint64 wrong_addr = r_stval();
    if (wrong_addr > p->sz) { // va is not allocated by sbrk()
        p->killed = 1;
        exit(-1);
    }
    if (wrong_addr < PGROUNDUP(p->trapframe->sp)) { // va is in or below the stack
        printf("under stack!\n");
        p->killed = 1;
        exit(-1);
    }
    uint64 lb = PGROUNDDOWN(wrong_addr);
    // printf("try alloc\n");
    char *mem = kalloc();
    if (mem == 0) {
        // printf("there is not enough free space\n");
        p->killed = 1;
    } else {
        memset(mem, 0, PGSIZE);
        // printf("memset\n");
        if (mappages(p->pagetable, lb, PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
            kfree(mem);
            uvmdealloc(p->pagetable, lb + PGSIZE, lb);
            p->killed = 1;
        }
    }
} 
```

> 处理 out-of-memory 的情况:

上面的代码已经处理了。

> 处理 parent-to-child 在 `fork` 时的内存复制的情况:

这里我想了很久才想明白，其实参考一下 `uvmunmap` 的处理就知道该怎么修改 `uvmcopy` 了，不过我一开始还是想歪了，直接是当检查到虚拟地址没有被物理地址映射时，手动给虚拟地址分配物理页，这样是有问题的，相当于 `fork` 的时候没有进行 lazy allocate 了。其实处理方案跟 `uvmunmap` 一摸一样，直接 `continue` 就行了，只要保证分配了 PP 的 VP 被复制过去就好了，没有分配 PP 的 VP，等它的虚拟地址被访问时，由 page fault 来处理就好了：

```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz) {
    pte_t *pte;
    uint64 pa, i;
    uint flags;
    // char *memold;
    char *mem;

    for (i = 0; i < sz; i += PGSIZE) {
        if ((pte = walk(old, i, 0)) == 0) {
            continue;
            panic("uvmcopy: pte should exist");
        }
        if ((*pte & PTE_V) == 0) {
            continue;
            panic("uvmcopy: page not present");
        }
        pa = PTE2PA(*pte);
        flags = PTE_FLAGS(*pte);
        if ((mem = kalloc()) == 0) {
            printf("no enough space for mem\n");
            goto err;
        }
        memmove(mem, (char *)pa, PGSIZE);
        if (mappages(new, i, PGSIZE, (uint64)mem, flags) != 0) {
            printf("can't mappages new\n");
            kfree(mem);
            goto err;
        }
    }
    return 0;

err:
    printf("uvmcopy err\n");
    uvmunmap(new, 0, i / PGSIZE, 1);
    return -1;
}
```

> 处理未分配 PP 的虚拟地址被直接作为系统调用的参数，在内核中被访问的情况：

这里当时理解错了实验提示的意思，另外就是，在内核中访问用户态的虚拟地址（通过 `copyin` 等函数），如果**该虚拟地址没有被分配 PP，是不会触发 page fault 的！因此，需要我们在访问这个虚拟地址之前，就手动给虚拟地址分配 PP**！

最好的时机就是在 `argaddr` 处，因为内核中，真正承担系统调用功能的 `sys_func` 函数都需要通过 `argaddr` 才能获取被作为参数传入到系统调用的虚拟地址，因此，我们可以在修改 `argaddr`，让它在获取传入的作为参数的虚拟地址之后，不马上返回，而是检查该虚拟地址是否被分配了 PP，如果没有，就检查该虚拟地址的合法性，如果合法，那么就给虚拟地址分配 PP：

```c
int argaddr(int n, uint64 *ip) {
    *ip = argraw(n);
    // ip 就是那个虚拟地址
    struct proc *p = myproc();
    if (walkaddr(p->pagetable, *ip) == 0) {
        uint64 wrong_addr = *ip;
        if (wrong_addr >= p->sz) { // va is not allocated by sbrk()
            return -1;
        }
        if (wrong_addr < PGROUNDUP(p->trapframe->sp)) { // addr 在 stack 的下方
            printf("under stack!\n");
            return -1;
        }
        uint64 lb = PGROUNDDOWN(wrong_addr);
        char *mem = kalloc();
        if (mem == 0) {
            printf("there is not enough free space\n");
            return -1;
        } else {
            memset(mem, 0, PGSIZE);
            if (mappages(p->pagetable, lb, PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
                kfree(mem);
                uvmdealloc(p->pagetable, lb + PGSIZE, lb);
                return -1;
            }
        }
    }
    return 0;
}
```

## 总结

到此其实也算做了 $5$ 个 Lab 了，差不多一半了，本课程的 Lab 代码量都不大，然后会有一两个比较难以想到的点，如果想到了，那么很容易就能写出来，想不到就抓瞎了。


