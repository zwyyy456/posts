---
title: "Linux 动态内存分配"
date: 2023-06-20T15:29:09+08:00
lastmod: 2023-06-20T15:29:09+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["csapp"]
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
## 动态内存分配器

进程中名为 heap 的 VM area 就是由动态内存分配器（dynamic memory allocator）来维护的。Heap 会向高地址（向上）增长。对每个进程，内核维护着一个名为 `brk` 的变量，该变量指向 Heap 的顶部，如下图所示：

![bjmDgoT7GqH14C8](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065824.png)

Allocator 将 Heap 视为一组不同大小的 block 组成的集合来维护。这里 block 和 chunk 的概念是等价的，该 block 要么是已分配的（allocated），要么是空闲的（free）。

>  Each block is a contiguous chunk of virtual memory that is either allocated or free. 

Allocator 可以分为显式分配器（explicit allocator）和隐式分配器（implicit allocator）。

- 显示分配器要求应用显示地释放任何已分配的块，分配也需要手动分配。例如 C 中的 `malloc` 和 `free`，C++ 中的 `new` 和 `delete`；
- 隐式分配器又被称为垃圾收集（garbage collection），例如 Java、C#。

## `malloc` 和 `free`

$32$ 位系统中，`malloc` 返回的地址总是 $8$ 的倍数，即 `malloc` 返回的地址的最低三位总是 $0$，亦即 `malloc` 分配的 block 至少占据 $8$ 的倍数个 byte；而 $64$ 位系统中，`malloc` 返回的地址总是 $16$ 的倍数。

即 $32$ 系统中，`malloc` 分配内存是 $8$ 字节对齐的；$64$ 位系统中，`malloc` 分配内存是 $16$ 字节对齐的。

## Allocator 的设计要求和目标

显式分配器必须在一些相当严格的约束条件下也能正常工作：

- **处理任意请求序列**：不能假设所有分配都有相匹配的释放请求等；
- **立即响应请求**：不允许分配器为了提高性能重新排列或者缓冲请求；
- **只使用 Heap**
- **对齐（aligning） block**：$32$ 位系统 $8$ 字节对齐，$64$ 位系统 $16$ 字节对齐；
- **不允许修改已分配的 block**：一旦 block 被分配就不允许修改或者移动，除非被释放（free）；

同时我们希望设计的分配器能**最大化内存利用率**并**最大化吞吐率**（即每秒能完成的分配或者释放请求数量）。

## 碎片（Fragmentation）

碎片分为内部碎片（internal fragmentation）和外部碎片（external fragmentation）。

**内部碎片**即要分配的 $size$ 只占分配的 $block size$ 的一部分的情况，例如分配一个 $2$ 字节的空间，由于要求至少 $8$ 字节对齐，那么必然有一部分用不到的空间为了满足对齐要求也被分配了出来。

**外部碎片**即空闲的空间总和大于要分配的 $size$，但是没有一个单独的 $block$ 的 $size$ 大到可以处理该分配请求。

![jpLhuXNnIB27kPO](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065826.jpg)

## 分配器实现

分配器实现过程中，我们需要考虑以下问题：

- *空闲 block 组织*：如何记录空闲 block？
- *放置*：如何选择一个合适的空闲 block来放置一个新分配的 block？
- *分割*：将一个新分配的块放置到某个空闲 block 之后，如何处理空闲 block 的剩余部分？
- *合并*：如何处理一个刚刚释放的 block（例如前或者后也有 free 的 block 呢）？

## 隐式空闲链表

一般来说，allocator 需要一些数据结构，从而区分 block 的边界并区分 allocated block 和 free block。大多数的 allocator 将信息嵌入这个块本身，就像 elf 文件有一个 elf header 一样，block 中也会有一个 header，存储了该 block 是否空闲，占据的空间 size 等信息。

![GC2LdqlwUK4k9WR](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065828.png)

假设 `malloc` 返回的指针是 `p`，由于 header 占据了 $32$ 个字节，header 本身是 `uint32_t` 类型的。那么 block 的起始地址 `start`：
```cpp
uint64_t start = (uint64_t)p - 4; // block 的起始地址
uint32_t header_val = *((uint32_t *)start); // header 的值
int allocated = header_val & 0x1; // 为 1 表示已分配，为 0 表示块仍然是空闲的
int size = header_val & 0xfffffffe; // 即将 header_val 的最后一位置 0
```

这种结构被称为**隐式空闲链表**，我们可以利用 block 的 header_val 中的 size 找到下一个块：`next = (uint64_t)start + size;`

隐式空闲链表的主要优点是简单，缺点则是：任何操作都可能需要遍历 heap 中的所有 block，时间复杂度是 $O(n)$ 的，此外还有个缺点是可能造成**内部碎片**。

## 显式空闲链表（Explicit Free List）

显式空闲链表是组织所有 free block 的显示数据结构，相比隐式空闲链表，显式空闲链表的每个空闲块中，都包含一个 `pred`（前驱）和 `succ`（后继）指针：如下图所示：

![dqZvAu57eCy2h36](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065830.png)

`pred` 和 `succ` 各占 $4bytes$。

在使用显式空闲链表后，首次适配（First fit）（即第一次找到大小满足的 free block），所需时间从 block 总数的线性时间减少到了 free block 数量的线性时间。释放一个 allocated block 所需时间也可以是常数的。

### 分离的空闲链表（Segregated Free List） -- 分离适配

分配器维护着一个空闲链表的数组，每个空闲链表是和一个大小类（size class）相关联的，