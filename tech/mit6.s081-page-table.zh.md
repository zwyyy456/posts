---
title: "MIT 6.S081 页表"
date: 2023-07-03T09:19:09+08:00
lastmod: 2023-07-03T09:19:09+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "mit", "os"]
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
## Paging hardware

总的来说，Xv6 的虚拟内存到物理内存的映射方式与 x64 是一致的，都是使用页表来进行映射。区别在于，Xv6 只使用了三级页表，而 x64 则是使用四级页表，另外，二者的页表层级的命名也有区别，对 Xv6 来说，最高级的页表是 L3（其地址存放于寄存器 satp 中）。

每个 page table 含有 512 个 page table entry，而每个 page table entry 的大小是 8KB，因此一个 page table 占据的大小正好是 4KB，即一个 VP 的大小！

![VMZ31LnYWRb8TUy](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065733.png)

标志位可以说是顾名思义，除了这个 dirty？（留待之后处理）

## Kernel address space

Xv6 中，每个进程都有属于自己的地址空间，以及一个全局唯一的描述内核地址空间的 page table，（kernel page table）。

内核内存布局：

QEMU 模拟了一台带 RAM（物理内存）的电脑，该 RAM 的起始地址是 $\text{0x80000000}$ ,结束地址至少是 $\text{0x86400000}$（该地址在 Xv6 中被定义为 `PHYSTOP`），这里的地址说的都是物理地址。

QEMU 会将设备接口以内存映射的控制寄存器暴露给系统软件，这些寄存器地址映射的内存都是在 $\text{0x80000000}$，即系统要访问这些设备，都是通过 $\text{0x80000000}$ 以下的物理地址直接访问，但通过这样的物理地址访问设备就不会经过 RAM 了。

内核空间的虚拟地址是**直接映射**到物理地址的，例如 `KERNBASE=0x80000000`，虚拟地址和物理地址都是这个值，可以理解为虚拟地址等于物理地址。

但是有几个内核虚拟地址不是直接映射的，如下图所示：

- trampoline page（蹦床页面），它映射在虚拟地址空间的顶部，user page table 具有相同的映射，即不论 kernel page table 还是 user page table，trampoline page 的虚拟地址都是在虚拟地址空间的顶部；
- kernel stack page，每个进程都有自己的 kernel stack，会映射到虚拟地址空间中比较高的那个 kernel stack，这样可以利用到 kernel stack 下方的那个 guard page。**Guard page is invalid!**（`PTE_V` 没有设置）。因此，一旦 kernel stack 发生溢出，就会发生异常。

![9iXhFnAN2DIozBx](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065735.png)

trampoline page 是 r-x 的，kernel stack 是 rw- 的。

从图上来看，其实只有 kernel data 和 kernel text 是直接映射的。
　　　
## Code: creating an address space 

关于操作 page table 和地址空间的代码主要位于 `kernel/vm.c` 中。

通过函数 `walk` 找到虚拟地址对应的 PTE 以及最终的物理地址（相当于之前 csapp 的虚拟内存那一章提到的 page walk）。

