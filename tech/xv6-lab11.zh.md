---
title: "Xv6 Lab11: Mmap"
date: 2023-08-03T13:53:24+08:00
lastmod: 2023-08-03T13:53:24+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "os", "lab", "mit"]
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
## 思路与实现

添加系统调用就不多说了。

整体流程应该是这样的，lab 的提示中，要求我们定义一个 vma 结构体，vma 的定义如下；然后 lab 的提示要求我们声明一个大小为 $16$ 的 vma 数组，并按需要从该数组分配，问题来了，数组在哪里声明呢？考虑到每个进程都有自己的虚拟地址空间，因此，每个进程都有自己的 virtual memory areas，要分配 vma 的时候，应该从每个进程自己的 vma 数组进行分配，于是，我们可以考虑为 `struct proc` 添加 `struct vma areas[NVMA]` 字段。

```c
struct vma {
    int fd;
    int rw_flag;
    uint64 start;
    uint64 cur;
    uint len;
    int state;
    int flags;
};

struct proc {
    // 已有的省略不写
    struct vma areas[NVMA];
};
```

在 `vma` 的定义中，`start` 表示起始地址，$[start, cur)$ 这一段虚拟地址（左闭右开）是已经绑定了 pp 的，pp 的数据与 file 绑定。

那么我们如何实现 `sys_mmap` 呢？这里可以参照 `sbrk`，递增 `p->sz`，然后仿照 `allocproc`，寻找状态为 `UNUSED` 的 vma，分配给本次 `sys_mmap`。注意如果文件本身是 read_only，并且以 `MAP_SHARED` 模式进行 map，那么 `flags` 不能为 `PROT_WRITE`，write only 的情况同理（即文件不可读）。

```c
uint64 sys_mmap(void) {
    uint len, offset;
    int prot, flags, fd;
    struct file *f;
    // 读取参数
    if (argint(1, (int *)&len) < 0 || argint(2, &prot) < 0 || argint(3, &flags) < 0 || argint(5, (int *)&offset) < 0) {
        return -1;
    }
    if (argfd(4, &fd, &f) < 0) {
        return -1;
    }
    struct proc *p = myproc();

    if (!f->writable && (prot & PROT_WRITE) && (flags & MAP_SHARED)) {
        return BADADDR;
    }
    if (!f->readable && (prot & PROT_READ) && (flags & MAP_SHARED)) {
        return BADADDR;
    }
    struct vma *area;
    for (area = p->areas; area < p->areas + NVMA; ++area) {
        if (area->state == UNUSED) {
            // 找到了空闲的 area
            area->state = USED;
            area->rw_flag = prot;
            area->flags = flags;
            area->start = p->sz;
            area->cur = p->sz;
            area->offset = offset;
            p->sz += len;
            area->len = len;
            area->f = f;
            filedup(area->f);
            return area->start;
        }
    }
    return BADADDR;
}
```

这里，我将 `sys_mmap` 的实现写在了 `sysfile.c` 中，这样就可以直接使用 `argfd` 来获取文件描述符和对应的 file 指针了，如果定义在 `sysproc.c` 中，那么通过 `p->ofile[fd]` 也可以获取文件描述符对应的 file 指针。

> 内核态中，虚拟地址到物理地址是直接映射的！除了 `PHYSTOP` 之上的部分。


这里我将 page fault 的下的处理封装成了一个函数 `mmap_handle`，出现 page fault，首先查看出现 page fualt 的虚拟地址是否位于 vma 中，如果是，得到对应的 vma 的指针 `area`，然后调用 `mmap_handle(wrong_addr, area, r_scause)`。

`mmap_handle` 的流程如下，首先判断 page fault 是否是我们所预期的，然后申请物理内存，map VP 与 PP（注意 `pte_flag` 不要漏掉 `PTE_U`），然后将文件的数据根据偏移量，读到对应的虚拟地址中去。读完之后，将 `area->cur` 和 `area->offset` 都增加读取的数据的字节数，同时减少 `area->len`，`area->len` 表示 vma 中还有多少空余空间。

```c
int mmap_handle(uint64 wrong_addr, struct vma *area, uint64 scause) {
    struct proc *p = myproc();
    if (scause == 13 && !(area->rw_flag & PROT_READ)) {
        printf("read a file that can't be read\n");
        return -1;
    }
    if (scause == 15 && !(area->rw_flag & PROT_WRITE)) {
        printf("write a file that is read-only\n");
        return -1;
    }

    char *mem = kalloc();
    if (mem == 0) {
        printf("without free mem\n");
        p->killed = 1;
    }
    memset(mem, 0, PGSIZE);
    uint64 lb = PGROUNDDOWN(area->cur);
    int pte_flag = PTE_U;
    if (area->rw_flag & PROT_READ) {
        pte_flag |= PTE_R;
    }
    if (area->rw_flag & PROT_WRITE) {
        pte_flag |= PTE_W;
    }
    if (area->rw_flag & PROT_EXEC) {
        pte_flag |= PTE_X;
    }
    uint area_pg = PGSIZE - (area->cur - lb);
    printf("area_pg: %p, cur: %p\n", area_pg, area->cur);
    area_pg = area_pg <= area->len ? area_pg : area->len;

    printf("mappages\n");
    if (mappages(p->pagetable, lb, PGSIZE, (uint64)mem, pte_flag) != 0) {
        printf("mappage failed\n");
        kfree(mem);
        uvmdealloc(p->pagetable, lb + PGSIZE, lb);
        return -1;
    }

    ilock(area->f->ip);
    if (readi(area->f->ip, 0, (uint64)mem + area->cur - lb, area->offset, area_pg) < 0) {
        printf("readi failed\n");
        kfree(mem);
        return -1;
    }
    iunlock(area->f->ip);

    area->offset += area_pg;
    area->cur += area_pg;
    area->len -= area_pg;

    return 0;
}
```

整体流程如下，首先

对于 `sys_munmap`，我们先读取参数 `addr` 和 `len`，然后执行 `unmap(addr, len);`，`unmap` 定义与 `proc.c` 中，在 `defs.h` 中声明即可，作业中已经说明了，要进行 `munmap` 的虚拟地址区间，要么左端点与 `area` 重合，要么右端点与 `area` 重合，如果左右端点都重合，说明整个文件都被解除 map 了，并且文件以 `MAP_SHARED` 方式 `mmap` 且可写，那么执行 `filewrite(area->f, addr, len)` 将这段虚拟内存的内容写回到文件中，直接全写回即可（反正作业中一次也不会 map 很多个 page）；写回之后，我们要执行 `fileclose(area->f)` 来递减 `f->refcnt`。

最后调用 `uvmunmap`，解除 VP 与 PP 的映射关系。

```c
uint64 unmap(uint64 addr, uint len) {
    struct vma *area;
    struct proc *p = myproc();
    for (area = p->areas; area < p->areas + NVMA; ++area) {
        // 左端点与 area 起始位置重合
        if (addr == area->start) {
            area->start -= len;
            break;
        }
        if (addr + len == area->cur + area->len) {
            area->cur -= len;
            area->len += len;
            break;
        }
    }
    if (area == p->areas + NVMA) {
        printf("no matached area\n");
        return -1;
    }
    if ((area->flags & MAP_SHARED) && (area->rw_flag & PROT_WRITE)) {
        // 写回到文件中
        filewrite(area->f, addr, len);
    }
    if (area->start == area->cur) {
        fileclose(area->f);
        area->state = UNUSED;
    }
    uvmunmap(p->pagetable, PGROUNDDOWN(addr), len / PGSIZE, 0);
    return 0;
}
```

我们还要修改 `exit` 和 `fork`。首先修改 `exit`，在 `exit` 时，遍历 `p->areas`，如果当前 area 不是 `UNUSED`，那么调用 `unmap` 解除当前 area 的映射。在 `fork` 中，如果父进程的 area 是 `USED`，那么我们需要将其中的数据拷贝给子进程。

```c
void exit(int status) {
    // 省略
    // Close all open files.
    for (int fd = 0; fd < NOFILE; fd++) {
        if (p->ofile[fd]) {
            struct file *f = p->ofile[fd];
            fileclose(f);
            p->ofile[fd] = 0;
        }
    }
    struct vma *area;
    for (area = p->areas; area < p->areas + NVMA; ++area) {
        if (area->state == USED) {
            unmap(area->start, area->cur - area->start);
        }
    }

    begin_op();
    iput(p->cwd);
    end_op();
}

int fork(void) {
    // 省略
    // increment reference counts on open file descriptors.
    for (i = 0; i < NOFILE; i++)
        if (p->ofile[i])
            np->ofile[i] = filedup(p->ofile[i]);
    np->cwd = idup(p->cwd);

    // 父进程与子进程具有相同 vma
    struct vma *area = p->areas;
    for (int i = 0; i < NVMA; ++i) {
        if (area[i].state == USED) {
            // np->areas[i].state = p->areas[i].state;
            // np->areas[i].rw_flag = p->areas[i].rw_flag;
            // np->areas[i].flags = p->areas[i].flags;
            // np->areas[i].start = p->areas[i].start;
            // np->areas[i].cur = p->areas[i].cur;
            // np->areas[i].offset = p->areas[i].offset;
            // np->areas[i].len = p->areas[i].len;
            // np->areas[i].f = p->areas[i].f;
            memmove()
            filedup(area->f);
        }
    }
    safestrcpy(np->name, p->name, sizeof(p->name));

    // 省略
}

```

最后，和 copy-on-write fork 一样，我们需要修改 `uvmunmap` 和 `uvmcopy`，如果发现 pte 不是 valid，跳过本次循环即可。

## 总结

之前看 csapp 的虚拟内存那一章的时候，书里面有提到 virtual memory area 的概念，当时不是很理解为什么需要一个这玩意，做了本次 lab 之后，对 linux 中的 `vm_area` 也算有所理解，本 lab 中，`vm_area` 是直接以数组形式存储的，每次都要 $O(n)$ 的时间才能找到对应的 area，而在 Linux 中，`vm_area` 是以红黑树的形式组织的，找到 vaddr 对应的 area，只需要 $O(\log n)$ 的时间。（当然，这也是因为我们只有 $16$ 个 area，而 Linux 中 area 的数量要多得多）。


