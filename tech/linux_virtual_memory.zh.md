---
title: "Linux 虚拟内存系统"
date: 2023-06-18T14:50:57+08:00
lastmod: 2023-06-18T14:50:57+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["csapp", "linux"]
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
## Linux 虚拟内存系统

首先，对 Linux 的虚拟内存系统做一个概述，以了解一个实际的操作系统是如何组织虚拟内存，以及如何处理缺页（page fault）的。

Linux 位为每个进程维护了一个单独的虚拟地址空间，形式如下：

![wX5QUzPJ9ldhGbj](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070011.png)

可以看到，虚拟地址空间可以分为内核虚拟内存空间和用户虚拟内存空间两部分，实际上，$64$ 位系统的虚拟空间划分是这样的：

![s4AYm5iEWwjGIHP](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070012.png)

我们可以看到，在用户内存空间和内核内存空间之间还有一大片的“未定义”的区域，这是为什么呢？（注意，后续图片将有灵魂画手出没！）。

之前我们提到，AMD 制定的 $64$ 位 CPU 架构时，虽然是 $64$ 位的，即总的虚拟地址空间是 $64$ 位的，但实际上，用到的虚拟地址其实只有其中的低 $48$ 位。

> 当我们把 `addr_val` 解释为一个虚拟地址时，我们使用的真正的虚拟地址，其实只有它的低 $48$ 位，（由 AMD 设计 CPU 架构的时候规定，其实 $48$ 位也完全够用了），后 $16$ 位的值会与 `addr_val` 的第 $47$ 位保持一致（全 $0$ 或者全 $1$），全 $0$ 表示该虚拟地址处于当前虚拟地址空间的用户态部分，全 $1$ 表示处于内核态部分。

换言之，虚拟地址的高 $16$ 位是由 CPU 在生成要访问的虚拟地址时，先生成低 $48$ 位的虚拟地址，再根据第 $47$ 位的值是 $0$ 还是 $1$，判断地址属于内核虚拟地址空间还是用户虚拟地址空间（或者说进程虚拟地址空间），再生成虚拟地址的高 $16$ 位。

如下图所示：

![ryE1jABVRiF2UXh](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070015.jpg)

## Linux 虚拟内存区域（area）

> Linux organizes the virtual memory as a collection of **areas** (also called segments). An area is a contiguous chunk of existing (**allocated**) virtual memory whose pages are related in some way. For example, the code segment, data segment, heap, shared library segment, and user stack are all **distinct areas**.

已分配的虚拟页必然存在于某个 area 中，换言之，不存在于任一 area 的虚拟页是不存在的，对应的虚拟地址是非法的！Heap 中可能存在有多个 area，这些 area 对应的 VP 都是堆上动态创建的数据的虚拟地址对应的 VP。

area 的存在，说明 Linux 系统允许虚拟地址空间有间隙，不存在的虚拟页不会占用内存、磁盘或者内核的任何额外资源。

下图是一个内核用来记录进程的虚拟内存区域的数据结构，这个数据结构存在于内核虚拟内存空间中。

![meYvPuhAEjgsTKk](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070015.png)

内核为系统中的每一个进程维护一个单独的 `task_struct`，`task_struct` 中的元素包含或者指向（即为指针）内核运行该进程所需要的所有信息（例如 PID、指向用户栈的指针、可执行目标文件的名字以及程序计数器 PC 等）。

`task_struct` 中的一个条目指向 `mm_struct`，`mm_struct` 描述了该进程的当前虚拟内存的状态，`mm_struct` 包含 `pgd` 和 `mmap` 两个字段，`pgd` 指向 PGD 的基地址，而 `mmap` 指向一个 `vm_area_structs` 的链表，该链表的每个 `vm_area_struct` 都描述了当前虚拟地址空间中的一个区域，一个 `vm_area_struct` 包含以下字段：

- `vm_start`：指向该区域的起始地址（应该是虚拟地址）；
- `vm_end`：指向该区域的结束处的地址；
- `vm_prot`：描述该区域包含的所有 VP 的读写许可权限；
- `vm_flags`：描述这个区域内的页面是与其他进程共享的，还是这个进程私有的（还描述了一些其他的信息）；
- `vm_next`：指向链表中下一个区域结构；

到这里，我们其实可以大致猜想进程的上下文切换时会发生什么，假设单核 CPU，从进程 $1$ 切换到进程 $2$，那么内核就会将 `task_struct2` 的 pgd 存放在当前 CPU 的 CR3 中，同时将 CPU 的 rip 寄存器更新为 `task_struct2` 中的 PC。

## Linux 缺页异常处理

假设 MMU 要翻译某个 `vaddr`，触发了一个 page fault。这个异常会导致控制转移到内核的缺页处理程序，处理程序随后会执行以下步骤：

1. `vaddr` 是否合法，即 `vaddr` 对应的 VP 是否存在于该进程的某个 `area` 中？因此缺页处理程序需要搜索 `vm_areas_structs` 链表，把 `vaddr` 和每个 `vm_area_struct` 的 `vm_start` 和 `vm_end` 进行比较，如果 `vm_start <= vaddr < vm_end`，那么说明该 `vaddr` 属于该 `vm_area_struct` 对应的 `area`，否则说明不属于。缺页处理程序会触发一个段错误，下图中标识为“1”，从而终止该进程；
    > 由于一个进程可以创建任意数量的 area，利用（`mmap`），所以实际上 Linux 对各个 area 构建了一棵 红黑树（RB-Tree），在这棵树上查找 `vm_area_struct`。类比 C++ STL 中的 map。

2. 假设 `vaddr` 合法，访问该 `vaddr` 是否合法，即该进程是否有权限？如果没有权限，例如不能写，或者用户进程试图访问内核虚拟内存，缺页处理程序会触发一个保护异常，从而终止该进程，下图中被标识为“2”；

3. 正常缺页：选择一个 victim page，如果该 victim page 为 dirty，就执行 `swap_out` 换出该 page，然后执行 `swap_in` 换入新的 page 并更新 PTE，然后缺页处理程序返回，CPU 重新执行导致缺页的指令，即读取 `vaddr`。

![5EXPJCDtwqIFd7Y](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070019.png)

## 内存映射（Memory Mapping）

许多进程包含同样的只读代码区域，例如，每个 C 程序都需要来自标准 C 库的诸如 `printf` 这样的函数，如果每个进程都在物理内存中存放这些常用代码的副本，那就是对内存的极大浪费。**内存映射**提供了一种用于控制多个进程如何共享对的象清晰的机制。

Linux 通过将一个磁盘上的对象（object）与一个 virtual memory area 关联起来，以初始化这个 virtual memory area，这个过程被称为**内存映射（memory mapping）**。

VM area 可以映射到两种文件类型的对象中的一种（Areas can be mapped to one of two types of objects）。（这里的映射我觉得翻译成关联似乎更合适）

1. *Regualr file in the Linux file system*：一个 VM area 被关联到一个 disk 的连续部分，例如一个可执行目标文件。File section 被划分成 VP 大小的的 pieces，每个 piece 包含一个 VP 的初始内容。这个 VP 就被称为 **file-backed page**。因此，文件的内容可以像内存一样被访问和操作，按需进行页面调度，这些 VP 在被 CPU 第一次引用之前，并没有被 swap_in 到物理内存中去。

2. *Anonymous file*：一个 VM area 也可以被映射到一个匿名文件，这个 VM area 中的 page 就被称为匿名页（Anonymous page）。当 CPU 第一次访问这个匿名页时，内核在物理内存中找一个合适的 victim page，如果 victim page 是 dirty 的，就将该 victim page 执行 swap_out，然后将该 PP 全置为 $0x0$，然后将 CPU 访问的这个 VP 对应的 PTE 的 `pte->present` 置为 $1$，说明该 VP 在物理内存中有对应的 PP 了。

## 内存反向映射（Reversed Mapping）

内存反向映射其实主要要解决三个问题：

- 当我们把一个 PP 换出到磁盘（swap space）时，我们要如何找到这个 PP 对应的 VP，从而更新该 VP 的 PTE 的 `pte->present`，并且
- 这个 PP 要换出到磁盘的何处；
- 当发生缺页中断后，要将 VP 对应的 PP 从磁盘（swap space）中 swap_in 到内存时，我们如何找到这个存在于磁盘中的“VP 对应的 PP”；

这里说的内存反向映射其实就是从 PP 到 VP 的一个映射：

在 yangminz 给出的 swap 和内存反向映射的简单 [实现](https://github.com/yangminz/bcst_csapp/blob/dedc042abefdbceba4feb495371354258ce49ca5/src/headers/memory.h) 中，添加了一个额外的 PP Descriptor 数据结构 `pd_t`，同时 `pte4_t` 中也添加了一个 `daddr` 字段，表示物理页要换出时，会换出到磁盘何处。

```cpp
typedef union
{
    uint64_t pte_value;

    struct
    {
        uint64_t present            : 1;    // present = 1
        uint64_t readonly           : 1;
        uint64_t usermode           : 1;
        uint64_t writethough        : 1;
        uint64_t cachedisabled      : 1;
        uint64_t reference          : 1;
        uint64_t dirty              : 1;    // dirty bit - 1: dirty; 0: clean
        uint64_t zero7              : 1;
        uint64_t global             : 1;
        uint64_t unused9_11         : 3;
        uint64_t ppn                : 40;
        uint64_t unused52_62        : 10;
        uint64_t xdisabled          : 1;
    };

    struct
    {
        uint64_t _present           : 1;    // present = 0
        uint64_t daddr            : 63;   // disk address
    };
} pte4_t;   // PT

// physical page descriptor
typedef struct
{
    int allocated;
    int dirty;
    int time;   // LRU cache

    // real world: mapping to anon_vma or address_space
    // we simply the situation here
    // TODO: if multiple processes are using this page? E.g. Shared library
    pte4_t *pte4;       // the reversed mapping: from PPN to page table entry
    uint64_t daddr;   // binding the revesed mapping with mapping to disk
} pd_t;

pd_t page_map[MAX_NUM_PHYSICAL_PAGE]; 
```

由这个数据结构，我们可以很容易的找到 PP 对应的 VP，毕竟 `pd_t` 中有 `pte4_t *pte4` 字段。

执行 `swap_out` 时，我们以 PP 的 `daddr` 作为实参执行 `swap_out(page_map[ppn].daddr, ppn);`，然后我们更新 victim pte 的字段内容，将 PP 的 `daddr` 赋给 `victim->addr`，并更新 `victim->present = 0`，表示该页表项对应的 VP 在物理内存中没有对应的 PP。然后执行 `swap_in`，`pte->daddr` 表示要换入的磁盘中的 PP 在磁盘中的地址，执行 `swap_in(pte->daddr, ppn)`。然后更新 PP 的 `daddr` 为新换入的磁盘中的 PP 的 `daddr`。

```cpp
static void page_fault_handler(pte4_t *pte, address_t vaddr) {
    // select one victim physical page to swap to disk
    assert(pte->present == 0);

    // this is the selected ppn for vaddr
    int ppn = -1;
    pte4_t *victim = NULL;
    uint64_t daddr = 0xffffffffffffffff;

    // 3. no free nor clean physical page: select one LRU victim
    // write back (swap out) the DIRTY victim to disk
    lru_ppn = -1;
    lru_time = -1;
    for (int i = 0; i < MAX_NUM_PHYSICAL_PAGE; ++ i) {
        if (lru_time < page_map[i].time) {
            lru_time = page_map[i].time;
            lru_ppn = i;
        }
    }

    assert(0 <= lru_ppn && lru_ppn < MAX_NUM_PHYSICAL_PAGE);

    ppn = lru_ppn;

    // reversed mapping
    victim = page_map[ppn].pte4;

    // write back
    swap_out(page_map[ppn].daddr, ppn);

    victim->pte_value = 0; // 将 victim pte 置零
    victim->present = 0;
    victim->daddr = page_map[ppn].daddr;

    // load page from disk to physical memory first
    daddr = pte->daddr;
    swap_in(daddr, ppn);

    pte->pte_value = 0;
    pte->present = 1;
    pte->ppn = ppn;
    pte->dirty = 0;

    page_map[ppn].allocated = 1;
    page_map[ppn].time = 0;
    page_map[ppn].dirty = 0;
    page_map[ppn].pte4 = pte;
    page_map[ppn].daddr = daddr;
}
```

这只是内存反向映射的一个非常非常简化的实现，实际上的内存反向映射是非常复杂的。如果 PP 对应的 VP 是 *file-backed page*，那么我们可以向上面的简化实现那样，得到 PP 对应的 VP，具体实现参加 [The object-based reverse-mapping](https://lwn.net/Articles/23732/)；如果 PP 对应的 VP 是 *anonymous page*，那么还要复杂很多，参加 [匿名反向映射的前世今生](https://richardweiyang-2.gitbook.io/kernel-exploring/00-index/01-anon_rmap_history) 和 [图解匿名反向映射](https://richardweiyang-2.gitbook.io/kernel-exploring/00-index/06-anon_rmap_usage)。

## 总结

到这里，我想我可以说对 Linux 虚拟内存系统有了一点很基础的了解。感谢 yangminz 大佬的 [视频讲解](https://space.bilibili.com/4564101/channel/seriesdetail?sid=983783)，如有不对，敬请指正。

## 参考

[CSAPP](https://csapp.cs.cmu.edu/)

[yangminz: bcst_csapp](https://github.com/yangminz/bcst_csapp)

[Kernel Exploring](https://richardweiyang-2.gitbook.io/kernel-exploring/00-index/06-anon_rmap_usage)


