---
title: "Xv6 Lab6: Copy-on-Write Fork for xv6"
date: 2023-07-17T13:42:27+08:00
lastmod: 2023-07-17T13:42:27+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "os", "lab", "mit"]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---
## 思路

经过 lab5: lazy page allocation 之后，对 xv6 的 page fault 的处理，算是有所了解了。

今天这个 COW 实验，在 2020 年的课程视频中有对思路的讲解，可以先看看 [课程翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec08-page-faults-frans/8.4-copy-on-write-fork)，厘清一下思路。

整体思路其实也不难，默认情况下，`fork` 会调用 `uvmcopy`，将父进程的 PP（物理页）复制一份，将这个 PP 的副本映射到子进程的 pagetable 的 VP（虚拟页）（子进程和父进程具有相同的虚拟地址，不同的 pagetable，不同的 PP，但是相同虚拟地址对应的 PP 的内容是一样的）。

> 我们这里讨论 vaddr、paddr 都是基于地址已经是 PGSIZE 对齐的情况来讨论的。

我们要做的修改就是，不再复制这个 PP，而是将 PP 的 paddr 同时映射到父进程的 vaddr 以及子进程的 vaddr。

在未修改 `uvmcopy` 之前，修改父进程的 vaddr 处的内容并不会影响子进程的 vaddr 处的内容，因为两个 vaddr 对应的是不同的 PP，只是 PP 的内容相同而已（在 pp 是 clean 的情况下）。

而修改了 `uvmcopy` 之后，写入父进程的 vaddr 会影响子进程的 vaddr 处的内容，这是我们不希望看到的，因此我们将这个 PP 对应的父进程的 pte 和子进程的 pte 的 PTE_W 位都清零，即标记为不可写，这样，当我们试图往这个 PP 中写入内容的时候，就会出现 page fault。

在出现 page fault 之后，我们需要在 `usertrap` 中处理这一情况，首先我们需要判断这个 page fault 是由于 Copy-on-Write Fork 这一机制导致的，而不是由于我们试图往一个真正的只读的页面写入内容，因此，我们需要用到 riscv 的 pte 的第 $8\sim9$ 位，这是给 riscv 给操作系统的设计者预留的，这里，如果 pte 的第 $8$ 位为 $1$，说明这是对应一个 Copy-on-Write Fork page。

而我们在 `usertrap` 中的处理方法也很简单，申请一块物理内存（一个新的 PP），将 PP 的内容复制到这个新的 PP 中去，然后解除 PP 和导致 page fault 的 vaddr 的映射，将这个新的 PP 映射到 vaddr。

此外，还有一个非常重要的细节，目前 xv6 中，除了 trampoline page 之外，一个 PP 同时只会映射一个用户进程的 VP，而我们这样修改之后，一个 PP 可能同时映射到父进程和它的子进程的 VP，因此父进程或者某个子进程退出时，正常情况下该 PP 会被释放，但是由于其他用户进程可能还需要访问该 PP，因此不能直接被释放。

因此我们需要一个引用计数数组 `ref_cnt`，每个 PP 对应 `ref_cnt` 中的一个元素，每次调用 `kfree` 时，将这个 PP 对应的的引用计数 $-1$，如果这样引用计数变为 0 了，就释放该物理页；如果引用计数在减一之前就已经是 $0$，也要释放该 PP。

> 我昨天做的时候就是卡在这里了，光想着复制 PP，重新映射的时候要递减原 PP 的引用计数，而 `kfree` 写成了引用计数不为 $0$ 就什么也不做，正常应该是不为 $0$ 的时候就将引用计数减一。

## 实现

首先，修改 `kalloc.c`，添加引用计数数组，这里我们参照 `kalloc` 中的 `kmem`，添加一个 `kref_cnt` 结构体：

```c
struct {
    struct spinlock lock;
    struct ref_arr[PHYSTOP / PGSIZE];
} kref_cnt;
```

由于每个物理页都需要一个引用计数，因此引用计数数组的大小是 `PHYSTOP / PGSIZE`，同时，由于可能涉及多进程同时写入 `ref_arr`，因此我们需要一个 `spinlock` 自旋锁来保护 `ref_arr` 的数据不出错。

添加操作 `ref_arr` 的辅助函数，包括递增、设值、递减、读取，记得在 `defs.h` 中添加 prototype。

```c
uint8 get_ref(uint64 paddr) {
    return kref_cnt.ref_arr[paddr / PGSIZE];
}

void dec_ref(uint64 paddr) {
    if (kref_cnt.ref_arr[paddr / PGSIZE] == 0) {
        return;
    }
    acquire(&kref_cnt.lock);
    kref_cnt.ref_arr[paddr / PGSIZE] -= 1;
    release(&kref_cnt.lock);
}

void inc_ref(uint64 paddr) {
    acquire(&kref_cnt.lock);
    kref_cnt.ref_arr[paddr / PGSIZE] += 1;
    release(&kref_cnt.lock);
}

void set_ref(uint64 paddr, uint8 val) {
    acquire(&kref_cnt.lock);
    kref_cnt.ref_arr[paddr / PGSIZE] = val;
    release(&kref_cnt.lock);
}
```

下一步则是修改 `kalloc` 和 `kfree` 两个函数，对于 `kalloc`，每次分配物理内存之后，将物理地址对应的引用计数置为 $1$ 即可，`kfree` 的逻辑稍微复杂一点，首先判断物理地址的引用计数是否大于 $1$，如果大于 $1$，就递减引用计数然后直接 `return`，否则将引用计数置为 $0$，然后执行释放物理内存的操作。

```c
void kfree(void *pa) {
    if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");
    if (get_ref((uint64)pa) > 1) {
        dec_ref((uint64)pa);
        return;
    }
    set_ref((uint64)pa, 0);
    struct run *r;

    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run *)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
}
void *kalloc(void) {
    struct run *r;

    acquire(&kmem.lock);
    r = kmem.freelist;
    if (r)
        kmem.freelist = r->next;
    release(&kmem.lock);

    if (r)
        memset((char *)r, 5, PGSIZE); // fill with junk
    set_ref((uint64)r, 1);
    return (void *)r;
}
```

再下一步是修改 `uvmcopy` 函数，修改思路前文已经给出了，同时要注意递增 paddr 对应的引用计数（因为子进程的 VP 也被这个 PP 映射了），同时**注意标志位的设置和清除**，`PTE_C` 需要自行定义在 `riscv.h` 中。

```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz) {
    pte_t *pte;
    uint64 pa, i;
    uint flags;
    // char *mem;

    for (i = 0; i < sz; i += PGSIZE) {
        if ((pte = walk(old, i, 0)) == 0)
            panic("uvmcopy: pte should exist");
        if ((*pte & PTE_V) == 0)
            panic("uvmcopy: page not present");
        *pte = *pte | PTE_C;
        *pte = *pte & (~PTE_W); // 设置 cow 标志位，PTE_W 清零
        pa = PTE2PA(*pte);
        flags = PTE_FLAGS(*pte);
        // if ((mem = kalloc()) == 0)
        //     goto err;
        // memmove(mem, (char *)pa, PGSIZE);
        if (mappages(new, i, PGSIZE, pa, flags) != 0) {
            kfree((char *)pa);
            goto err;
        }
        inc_ref(pa); // 递增引用计数
    }
    return 0;

err:
    uvmunmap(new, 0, i / PGSIZE, 1);
    return -1;
}
```

然后就是修改 `usertrap` 了，先判断是否是 page fault 引起的 trap，方法与 [lab5](https://blog.zwyyy456.tech/zh/posts/tech/xv6-lab5/) 一致，然后通过 `if ((*pte & PTE_W) == 0 && (*pte & PTE_C) == PTE_C)` 判断发生 page fault 是不是“向 cow page 写入” 造成的。如果是，执行 `remappage`（定义在 `vm.c` 中）。

`remappage` 的主要内容就是解除原先的 PP 对 vaddr 的映射，将申请的新 PP 映射到 vaddr，同时注意处理引用计数。

```c
} else if (r_scause() == 13 || r_scause() == 15) {
    // copy on write
    uint64 vaddr = PGROUNDDOWN(r_stval());
    uint64 paddr;
    pte_t *pte;
    if ((paddr = walkaddr(p->pagetable, vaddr)) == 0) { // 判断该 vaddr 是否有效，并获取 vaddr 对应的 paddr
        p->killed = 1;
        exit(-1);
    }
    // 检查 PTE_W 和 PTE_C
    pte = walk(p->pagetable, vaddr, 0);
    if ((*pte & PTE_W) == 0 && (*pte & PTE_C) == PTE_C) {
        if (remappage(p->pagetable, vaddr, paddr, pte) == 0) {
            p->killed = 1;
            exit(-1);
        }
    }
} 
```

```c
uint64 remappage(pagetable_t pagetable, uint64 vaddr, uint64 old_pa, pte_t *pte) {
    if (get_ref(old_pa) == 1) {
        // 说明只有一个进程在使用这个 PP 了，直接修改 pte 的 PTE_W 位即可
        *pte = *pte | PTE_W;
        *pte = *pte & (~PTE_C);
        return old_pa;
    }
    char *mem;
    if ((mem = kalloc()) == 0) {
        return 0;
    }
    memmove(mem, (char *)old_pa, PGSIZE);
    // 记得先解除映射，否则会 "panic, remap!"
    uvmunmap(pagetable, vaddr, 1, 1); // uvmunmap 的时候就会调用 free，递减引用计数
    // dec_ref(old_pa);
    if (mappages(pagetable, vaddr, PGSIZE, (uint64)mem, PTE_W | PTE_U | PTE_X | PTE_R) != 0) {
        kfree(mem);
        return 0;
    }
    return (uint64)mem;
}
```

修改 `copyout` 的步骤与修改 `usertrap` 基本一致，只是要注意一点：在向 cow page 写入时，我们是重新申请了 PP，将原先的 PP 的内容复制到新的 PP，并执行了 `remappage`，那么最后向物理地址写入数据的时候，应该向我们新申请的 PP写入！

```c
int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len) {
    uint64 n, va0, pa0;

    while (len > 0) {
        va0 = PGROUNDDOWN(dstva);
        pa0 = walkaddr(pagetable, va0); // pa0 一定是 PGSIZE 对齐的
        uint64 mem = pa0;
        if (pa0 == 0) {
            return -1;
        }
        // copy on write
        pte_t *pte;
        // 检查 PTE_W 和 PTE_C
        pte = walk(pagetable, va0, 0);
        if ((*pte & PTE_W) == 0 && (*pte & PTE_C) == PTE_C) { // 说明要写 COW，否则无异常
            if ((mem = remappage(pagetable, va0, pa0, pte)) == 0) {
                return -1;
            }
        }
        n = PGSIZE - (dstva - va0);
        if (n > len)
            n = len;
        memmove((void *)(mem + (dstva - va0)), src, n);

        len -= n;
        src += n;
        dstva = va0 + PGSIZE;
    }
    return 0;
}
```

## 总结

这个实验与上一个 lazy allocation 的实验大同小异，都能很好加深对 page fault 异常的处理的理解。