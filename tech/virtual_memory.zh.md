---
title: "虚拟内存"
date: 2023-07-28T19:55:47+08:00
lastmod: 2023-07-28T19:55:47+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["csapp", "linux"]
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
## 虚拟地址空间与物理地址空间

地址空间（address space）是一个非负整数地址的有序集合：

$$\lbrace 0, 1, 2, \cdots\rbrace$$

如果地址空间中的整数是连续的，那么我们说它是个**线性地址空间（linear address space）**，为了简化讨论，我们总是假设我们使用的是线性地址空间。

地址空间的大小由表示最大地址所需的位数来决定，例如 $N - 1= 2^n - 1$，因此最大地址需要 $n$ 位数来表示，于是一个包含 $N = 2^n$ 个地址的地址空间就叫一个 $n$ 位地址空间。

### 虚拟地址空间

在一个带虚拟内存的系统中，CPU 由 $N=2^n$ 的地址空间中生成虚拟地址，这个虚拟地址的有序集合称为虚拟地址空间（virtual address space）：

$$ \lbrace 0, 1, 2, \cdots N - 1\rbrace$$

这个地址空间是 $n$ 位的。

### 物理地址空间

一个系统还有一个**物理地址空间（physical address space）**，对应于系统中物理内存的 $M$ 个字节：

$$\lbrace0,1,2,\cdots,M-1\rbrace$$

$M$ 并不要求是 $2$ 的幂，例如可能是 $12GB$，但是为了简化讨论，我们假设 $M=2^m$，即地址空间是 $m$ 位的。

## 物理内存作为虚拟内存的缓存

概念上而言，虚拟内存被组织为一个存放在磁盘上的由 $N=2^n$ 个连续字节大小的单元组成的数组，每个字节都有一个唯一的虚拟地址，作为到数组的索引；对应的，计算机的主存（main memory，后面简称内存）被组织成一个由 $M=2^m$ 个连续的字节大小的单元组成的数组，每字节都有一个唯一的物理地址。

VM（Virtual Memory）系统通过将虚拟内存分割为称为虚拟页（Virtual Page，VP）的大小固定的块来处理这个问题，每个虚拟页的大小为 $P=2^p$ 字节。相应的，物理内存被分割为物理页（Physical Page，PP）来处理，大小也为 P 字节。这里的物理页就像是 SRAM cache 中的 block。

虚拟页有三种可能的状态（同一 VP 只会存在一种状态）：

- 未分配的：VM 系统还没有分配（或者创建）的页。未分配的 VP 没有任何数据与它关联，因此不占据任何磁盘空间；
- 缓存的：当前已缓存在物理内存中的已分配页；
- 未缓存的：未缓存在物理内存中的已分配页；

![Vv1O4hYLMq5ecQN](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065753.png)

我们使用 DRAM cache 来表示 VM 系统的 cache，用 SRAM cache 来表示 CPU 和 内存之间的 L1、L2、L3 cache。DRAM cache 用于在主存中缓存虚拟页。

可以这样理解，**VP** 是存在于磁盘中的，VP 的 **cache** 即 **PP** 才存在于主存中。如果说主存中存在 VP，那么这个 VP 实际上是磁盘中的 VP 对应的 PP。并且，由于磁盘访问速度，比起内存来说都非常非常慢，因此**缺页（page fault）**（即 DRAM cache miss）的代价比 SRAM cache miss 的代价要昂贵得多得多。因此 VP 和 PP （即 dram cache block）往往很大（$4KB$），并且 DRAM cache 是全相联的，即任何 VP 都可以放置在任何 PP 中。

-  DRAM cache 总是采用 write back 而不是 write through。

## 页表（page table）

页表是一种存放在物理内存中的数据结构，用来确定主存中是否存在与磁盘上的 VP 对应的 PP，如果存在，就能通过页表找到这个 PP。页表负责将 VP 映射到 PP。

页表其实就是一个 Page Table Entry（PTE）的数组，每个 PTE 是由一个有效位（valid bit）和一个 $k$ 位地址字段组成的。有效位表明了该 VP 是否被缓存在 DRAM 中。如果有效位为 $1$，$k$ 位地址字段就表示 DRAM 中相应的物理页的起始地址，结合页内偏移（page offset）就能得到虚拟地址（VA）对应的物理地址（PA）。

![OEGDnNvqe85zIUi](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065754.png)

页表可以包含很多个页表项，我们先讨论只有一级页表的情况，假设一个虚拟地址空间是 $32$ 位的，一个页的大小是 $4KB$，那么一共需要 $\frac{2^{32}}{2^2*2^{10}}$ 个 PTE，即 $1M$ 个（$M$ 指兆）。

### 一级页表的页命中

如下图，假设 CPU 要读取 VA 中的一个字，VA 对应的 VP 是 VP 2，假设 VP 2 被缓存在 DRAM 中，那么 MMU 会将 VA 作为一个索引来定位 PTE 2，使用 PTE 中的物理内存地址（指向 VP 2 对应的 cache —— PP 1）的起始地址，结合页内偏移即可构造出虚拟地址对应的物理地址。

![kw3HF54UfYsnKS7](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065756.png)

### 一级页表下的缺页

在虚拟内存的说法中，DRAM cache miss 被称为缺页（page fault）。由下图，假设 CPU 要读取 VA 中的一个字，VA 对应的 VP 是 VP 3，VP 3 可以由对应 PTE 3 的有效位确定它没有被缓存在主存中，然后就会触发一个缺页异常。

![v3Sz9MZsKTakdPw](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065757.png)

发生缺页异常后，事件流程如下：

缺页异常调用内核中的缺页异常处理程序，该程序选择一个牺牲页（victim page），此例中我们指定为 PP3 中的 VP4，如果 PP3 中的 VP4 是 dirty 的，那么内核将 VP4 复制回磁盘，修改 VP4 的 PTE，说明 VP4 不再缓存在 DRAM 中；然后内核从磁盘复制 VP3 到内存中的 PP 3，更新 PTE3，然后内核返回。 当异常处理程序返回时，会重新启动导致缺页的指令（即读取 VP3 中的字），这时就可以不发生缺页异常而正常读取了。如下图：

![Py9TRBiVbXzm1np](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-65801.png)

### 虚拟内存与 SRAM cache

我们可以发现，虚拟内存系统与 SRAM cache 在概念和设计上有许多相似的地方，SRAM cache 中的 block 在 VM 系统中被称为页（page），磁盘和内存之间传送页的活动叫做交换（swapping）或者页面调度（paging）。直到发生缺页才进行页面换入内存（swap in）的策略称为按需页面调度（demand paging）。

局部性（locality）保证了此种机制下的 VM 系统的工作效率。

## 虚拟内存的意义

1. **简化链接**：每个进程的虚拟内存结构都使用相同的基本格式，代码段总是从虚拟地址 $0x400000$ 开始，链接器在设计和实现时无需考虑代码和数据实际会存放在物理内存的何处，大大简化了链接器的设计和实现。同时避免了同一个物理地址被不同虚拟地址映射到时的冲突现象。

2. **简化加载**：每个页被初次引用时，VM 系统会自动按照需要调用数据页。TODO(zwyyy)

3. **简化共享**：操作系统通过将不同进程的中适当的 VP 映射到相同的 PP（例如 C 标准库的程序对应的 VP，又或者内核对应的 VP），从而使得多个进程共享这个标准库的同一个副本，而不是每个进程都包含一个单独的副本

4. **简化内存分配**：例如创建数组需要分配一段连续的内存空间，我们只需要分配一段连续的 VM 即可，而不需要分配连续的 PM，即这些分配的连续的 VM 对应的 PM 很可能是不连续的。

5. **实现内存保护**：通过在 PTE 上添加一些额外项，可以实现对一个 VP 的内容的访问权限控制。

## 地址翻译

形式上说，地址翻译是一个 N=2^n 元素的虚拟地址空间（VAS）到 M=2^m 元素的物理地址空间（PAS）中元素之间的映射：
    
$$MAP:VAS \rightarrow PAS \cup \emptyset$$

$$ MAP(A)=\left\lbrace \begin{array}{l} \emptyset\ \ 如果虚拟地址\ A\ 的数据不在物理内存中 \newline A'\ \ 如果虚拟地址\ A\ 处的数据在\ PAS\ 的物理地址\ A'\ 中\end{array}\right. $$

下图表示了 MMU 如何利用页表实现这一映射：CPU 中有一个 control register，又称页表基址寄存器（Page Table Base Register，PTBR）指向当前页表，注意不是 PTE，PTE 要由 VPN 才能确定！

![MVan5f6Wl8dc9kE](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065802.png)

n 位的虚拟地址包含两个部分:

- 一个 p 位的虚拟页面偏移（Virtual Page Offset，VPO）；
- 一个（n-p）位的虚拟页号（Virtual Page Number, VPN）。

MMU 利用 VPN 来选择适当的 PTE ，例如 VPN 为 0 选择 PTE 0，VPN 为 1选择 PTE 1 ，以此类推（实际上不一定是这样，为了表达意思这样说）。将页表条目中的物理页号（Physical Page Number, PPN）和虚拟地址中的 VPO 串联起来，就得到了物理地址。

由于这里 PP 和 VP 大小相同，都是 $P = 2^p$，因此 VPO 和 PPO 都是一样的。

页面命中时，CPU 硬件执行的步骤如下：

- 处理器生成或者说要访问一个虚拟地址（VA），并把它传递给 MMU；
- MMU 计算得到 PTE 地址（简单位运算即可），从 DRAM 或者 DRAM 的 cache（SRAM）中读取到这个 PTE；
- DRAM 或者 SRAM 向 MMU 返回 PTE；
- MMU 由 PTE 构造得到物理地址，并把它传送给 DRAM 或者 SRAM；
- DRAM 或者 SRAM 将所请求的字返回给处理器。

发生 page fault 时，相比页面命中多了缺页异常的发生与处理步骤。注意，当缺页异常完成处理后，CPU 会再次执行导致缺页的指令，即读取那个 PTE，此时会正常发生页面命中了。

## 多级页表

使用多级页表，相比只使用一级页表，能大大减少页表本身需要占据的物理内存中（这是因为未分配的页不会创建对应的 PTE）。

讲完多级页表的结构之后，就能理解为什么多级页表能减少页表本身的物理内存占用了。

下图描述了使用 $k$ 级页表层次结构的地址翻译：

- 考虑一个使用 $k$ 级页表结构层次的地址翻译，虚拟地址被划分为 $k$ 个 VPN 和一个 VPO，每个 VPN i 都是到 $i$ 级页表的页表项的索引，其中 $1 \leq i \leq k$；
- 对 $1 \leq j \leq k - 1 的每个 j$，第 $j$ 级页表中的每个 PTE 都指向第 $j + 1$ 级的某个页表的基址，相当于 PTE 中是基地址，VPN i  是相对于基址的偏移。

![6uIk9sdzjPFxSqg](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065803.png)

如果一级页表中的一个 PTE 是空的，那么相应的二级页表根本不会存在，也不会占用物理内存。而对于一个典型的程序，$4GB$ 的虚拟地址空间的大部分都是未分配的，因此可以大大节省物理内存占用。

例如：假设只有一级页表，那么对于 $4GB$ 的虚拟地址空间，每个进程需要 $\frac{2^32}{4 * 2^{10}} * 4B = 2^{20} * 4B = 4 MB$ 的空间（$B$ 表示字节，Byte），无论虚拟地址空间实际上使用了多少。一级页表有 $2^{10}$ 个页表项，二级页表也是 $2^{10}$ 个页表项，假设虚拟地址空间使用了 $20$%，那么实际上占用的物理内存为 $2^{10} * 4B + \frac{1}{5} * 2 ^{20} * 4B \approx 0.804 MB$，大大节省了内存空间，而四级页表会节省得更多。

$64$ 为虚拟地址空间的思考方式同理，注意 $64$ 位虚拟地址空间的情况下，单个 PTE 占据 $8B$；而 $32$ 位是 $4B$。

在这里可以给出简单描述虚拟地址的结构体：

```cpp
typedef union {
    uint64_t address_value;
    struct {
        uint64_t co : SRAM_CACHE_OFFSET_LENGTH; // 12
        uint64_t ci : SRAM_CACHE_TAG_LENGTH; // 4
        uint64_t ct : SRAM_CACHE_INDEX_LENGTH; // 32
    };

    // physical address: 16 bits
    struct {
        /* data */
        union {
            uint64_t paddr_value : PHYSICAL_ADDRESS_LENGTH; // 16
            struct {
                uint64_t ppo : PHYSICAL_PAGE_OFFSET_LENGTH; // 12
                uint64_t ppn : PHYSICAL_PAGE_NUMBER_LENGTH; // 4
            };
        };
    };
    // virtual address: 48b bits
    struct {
        /* data */
        union {
            uint64_t vaddr_value : VIRTUAL_ADDRESS_LENGTH; // 48
            struct {
                uint64_t vpo : VIRTUAL_PAGE_OFFSET_LENGTH;  // vpo 与 ppo 是一致的, 12 bits
                uint64_t vpn4 : VIRTUAL_PAGE_NUMBER_LENGTH; // 9 bits 
                uint64_t vpn3 : VIRTUAL_PAGE_NUMBER_LENGTH; // 9 bits
                uint64_t vpn2 : VIRTUAL_PAGE_NUMBER_LENGTH; // 9 bits
                uint64_t vpn1 : VIRTUAL_PAGE_NUMBER_LENGTH; // 9 bits
            };
        };
    };
    struct {
        uint64_t tlbo : TLB_CACHE_OFFSET_LENGTH; // virtual page offset, 12 bits
        uint64_t tlbi : TLB_CACHE_INDEX_LENGTH;    // tlb set index, 4 bits
        uint64_t tlbt : TLB_CACHE_TAG_LENGTH;  // TLB line tag, 32 bits
    };
} address_t;
```

这里利用 C 语言中的 union 这一类型，正好可以表达我们如何看待给出的一个 $64$ 位地址 `addr_val`（即 $64$ 位的 `uint64_t` 类型的数）。

当我们把 `addr_val` 解释为一个虚拟地址时，我们使用的真正的虚拟地址，其实只有它的低 $48$ 位，（由 AMD 设计 CPU 架构的时候规定，其实 $48$ 位也完全够用了），后 $16$ 位的值会与 `addr_val` 的第 $47$ 位保持一致（全 $0$ 或者全 $1$），全 $0$ 表示该虚拟地址处于当前虚拟地址空间的用户态部分，全 $1$ 表示处于内核态部分。

对 `addr_val` 来说，最低的 $12$ 位表示 Virtual Page Offset（VPO），它与对应的物理地址的 Physical Page Offset（PPO）是完全相等的。高 $39\sim 47$ 这 $9$ 位的 vpn1 代表 Page Global Directory（PGD）的 PTE 的索引，从而可以知道这个 `addr_val` 对应着 PGD 中的哪个 PTE，PGD 中的 PTE 存放的是 Page Upper Directory（PUD）的基地址；$30\sim 38$ 位的 vpn2 表示 PUD 的 PTE 的索引，PUD 中的 PTE 存放着 Page Middle Directory（PMD）的基地址；$21\sim 29$ 位的 vpn3 表示 PMD 的 PTE 的索引，PMD 的 PTE 中存放着到 Page Table（PT）的基地址；$12\sim 20$ 位的 vpn4 表示 PT 的 PTE 的索引，而 PT 的 PTE 中存放着 `addr_val` 对应的 PP 的页号，即 PPN，由 PPN 和 PPO（$PPO=VPO$） 即可拼接成物理地址 `paddr`。这个 **`page_walk`** 的流程如下图：

![aVIQz4i1eX68rT3](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065805.png)

对每个进程来说，其 PGD 的起始地址由 CPU 的 CR3 Control Register 给出，CR3 指向 PGD 的起始位置，CR3 的值是每个进程上下文的一部分，每次上下文切换时，CR3 的值都会恢复。从 PGD 的最高位就能看出来是对应的虚拟地址是属于 user 还是 kernel 的。

这也就解答了为什么不同进程的同一个 `vaddr` 会映射到不同的 `paddr`，因为它们最一开始指向的 PGD 就不一样了！

## PTE 格式

PGD、PUD、PMD 的 PTE 格式如下：

![NrF5Jt1IZRsHOdz](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065807.png)

地址段包含一个 $40$ 位的物理页号（PPN），$40$ 位的物理页号即要求物理页表 $4KB$ 对齐（$4k = 2^{12}$）。

PT 的 PTE 格式如下：

![xc7LrVulNSFm6n1](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065809.png)

R/W 位可以确定页的内容是可以读写的还是只读的，U/S 确定是否可以在用户模式访问该页；D 位称为 dirty bit（类似 cache 中的 dirty 状态）。

## TLB

MMU 中存在一个关于 PTE 的小的缓存，称为 TLB，即拿到 `vaddr` 后，不需要一步步 page walk 得到最终的 PTE，可以直接从 TLB 中拿出 PT 的 PTE。

TLB 的结构与 SRAM cache 很像：

```cpp
typedef struct {
    int valid;
    uint64_t tag;
    uint64_t ppn;
} tlb_cacheline_t;

typedef struct {
    tlb_cacheline_t lines[NUM_TLB_CACHE_LINE_PER_SET];
} tlb_cacheset_t;

typedef struct {
    tlb_cacheset_t sets[1 << TLB_CACHE_INDEX_LENGTH];
} tlb_cache_t;

static tlb_cache_t mmu_tlb;

static int read_tlb(uint64_t vaddr_value, uint64_t *paddr_value_ptr, int *free_tlb_line_index) {
    address_t addr;
    addr.address_value = vaddr_value;

    tlb_cacheset_t *set = &mmu_tlb.sets[addr.tlbi]; // 获取 tlb 的 set 的 index
    for (int i = 0; i < NUM_TLB_CACHE_LINE_PER_SET; ++i) {
        tlb_cacheline_t *line = &set->lines[i]; // cache line 一行行匹配
        if (line->valid == 1) { // 这里为 1 表示 invalid
            *free_tlb_line_index = i; 
        }
        if (line->tag == addr.tlbt && line->valid != 1) { // 说明 TLB 中存在着 PTE 的 cache
            *paddr_value_ptr = line->ppn;
            return 1;
        }
    }
    paddr_value_ptr = NULL;
    return 0;
}
```

## 总结

何为 `page_walk`？即由 `CR3->PGD->PUD->PMD->PT->PTE`，最后由 PTE 得到 PPN，再由 PPN 和 PPO（VPO）得到物理地址。

理解了 `page_walk`，我认为就可以说初步理解了这个虚拟内存系统。



