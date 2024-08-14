---
title: "MIT 6.S081 File system performance and fast crash recovery"
date: 2023-07-27T16:34:51+08:00
lastmod: 2023-07-27T16:34:51+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["linux", "mit", "os", "xv6"]
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
## 引入

当我们针对文件系统讨论 logging 或者 journal 时，其实是在讨论同一件事，二者是同义词。

这一部分主要是讨论 Linux 的 ext3 文件系统，它相比 ext2，可以就说就是加了一层 logging，其他基本没有改变。我们要关注 ext3 与 xv6 的文件系统的不同之处，重点放在 ext3 是如何在保证 logging 的同时尽可能提升性能的。

## ext3 file system log format

ext3 数据结构与 xv6 相似，在内存中存在 block cache，它们是 write-back 的（即改动不会马上写回到磁盘）。

logging 系统有两个非常重要的准则：

1. **write-adead rule**：必须现在 log 中记录好所有这些写操作，才能将这些写操作应用到磁盘；
2. **freeing rule**：即我们不能覆盖或者重用 log。

ext3 还维护了一些 transaction 的信息，`transaction_t` 中包含：

- 一个序列号；
- 一系列该 transaction 修改的 block 的编号，这些 block 编号指向 cache 中的 blcok；
- 一系列的 handle，handle 对应属于transaction 的系统调用，它们会读写 cache 中的 block；

ext3 的磁盘组织与 xv6 类似，存在一个文件系统树，包含 inode、目录、file 等，存在 bimtap lock 来标识每个 data block 是被分配还是空闲的，在磁盘的一个指定区域保存 log。

ext3 的 log 与 xv6 不太一样，在 ext3 的 log 中，也存在着一个 super block（注意这是 log 的 blcok，而不是文件系统的 block），log 的 superblock 包含了 log 中第一个有效的 transaction 的起始位置和序列号。在 log 中，除了 super block 以外的 block 存储了 transaction，每个 transaction 在 log 中包含了：

- 一个 description block，其中包含了 log 中的数据对应的实际 block 编号，有点像 xv6 中的 log 的 header block；
- 针对上面每一个 block 编号的更新数据；
- 当一个 transaction 完成并 commit 了，会有一个 commit block；

log 中可能存在多个 transaction 的 block，如下图所示：

![CsyKnZk7MVm2TQH](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070023.png)

log 的 superblock 中的起始位置和序列号属于**最早的、排名最靠前，也是最有效的 transaction**。

在 crash 之后恢复过程中会扫描 log，为了将 descriptor block 和 commit block 以及 data block 区分开来，descriptor block 和 commit block 会以一个 $32$ bit 的 magic nubmer 为起始，有点像 elf 文件，用于帮助软件区分不同的 block。

log 中可能有多个 transaction，但同一时间只有一个正在进行的 transaction，这个 transaction 对应的系统调用只会更新 cache 中的 block；当 ext3 决定结束当前正在进行的 transaction，它会做两件事：

1. 开始一个新的 transaction 作为当前 transaction 的下一个 transaction；
2. 将当前刚刚完成的 transaction 写入到磁盘，完成 commit，此时磁盘的 log 中存放了对磁盘 data blocks 的更新的记录，但是还没有讲更新应用到磁盘上，磁盘 log 分区上可能存在一系列这样的 transaction，然后内存中有一个正在进行，还没有 commit 的 transaction。

log 是一个循环数据结构，如果用到了 log 分区的最后，logging 系统会从 log 的最开始位置开始使用。

## ext3 如何提升性能

ext3 主要通过三个方面提升性能：

1. **asynchronous**：它提供异步（asynchronous）系统调用，系统调用写入到磁盘之前就返回了，系统调用只会更新缓存在内存中的 block，不用等待写磁盘操作，但它可能会等待读磁盘；
2. **batching**：它提供了批处理（batching）的能力，可以将多个系统调用打包成一个 transaction；
3. **concurrency**。

> xv6 也提供了有限能力的 **batching**。

### 异步系统调用

异步系统调用说明**系统调用修改了 cache block 就立即返回，而不会触发写磁盘**，因此系统调用可以快速返回，此时应用程序可以继续进行计算，而文件系统在后台完成之前系统调用所需的写磁盘操作，这被称为 **I/O concurrency**。同步系统调用中，应用程序需要等磁盘写入完成之后，才能从系统调用中返回。

异步系统调用还使得 batching 变得容易实现；

> 这个大概就像 xv6 中一样，执行系统调用，写入 cache block 之后就返回了，之后文件系统会批量将这些系统调用影响的 cache block 写回磁盘；

异步系统调用也有一个缺点，即系统调用的返回，并不能表示其工作真的完成了。例如，我创建一个文件，像文件写数据，然后关闭文件，并在终端输出 done。在终端看到了 done，并不意味着文件的数据就已经都写入到磁盘了，例如这个流程需要 `open`、`write`、`close` 系统调用，这三个系统调用写入了 cache block 之后就返回了，然后终端输出 done，文件系统在后台写入 log，再 commit 并 install trans，在输出 done 时，文件系统很可能还没有完成工作（还没有完成 commit，将修改记录到 log），如果此时断电了，那么尽管程序显示已经完成了创建和写入，但可能我们重启之后在磁盘中找不到这个文件。

文件系统对类似数据库或者文件编辑器之类的程序，提供了一个名为 `fsync` 的系统调用，来确保 crash 之后会有一个预期的结果。这个系统调用接受一个文件描述符作为参数，它会告诉文件系统去完成所有与该文件相关的写磁盘操作，在所有数据都写入到磁盘之后，`fsync` 才会返回。TODO(zwyyy)

### batching

在任何时候，ext3 只会有一个存在于内存的 transaction，一个 transaction 可能包含多个不同的系统调用，ext3 的工作方式类似下面的描述：

ext3 首先宣告要开始一个新的 transaction，接下来一段时间（例如 $5$ 秒）内所有的文件系统调用都是这个 transaction 的一部分，这些系统调用打包在一个 transaction 中，在 $5$ 秒钟结束的时候，ext3 会 commit 这个包含了可能有数百个更新的大 transaction。

batching 的优势可以分为以下三点：

1. 它将一次 transaction 的固有的开销平摊给了多次系统调用，固有开销涉及写 transaction 的 descriptor block 和 commit block，在机械硬盘中，这需要查找 log 的位置，旋转磁头，这个成本是很高的（固态硬盘会好一些），batching 使得我们只需要对一批系统调用执行一次，而不用对每个系统调用都执行一次；
2. 更容易触发 write absorption，即**多次系统调用，但只用向对应的 data block 或者 bitmap block 写一次（向 block cache 可能写入多次）**。例如创建一堆很小的文件（几 kB 或者几十 kB），这些 inode 的 data block 可能在几个连续的甚至同一个 data block；而如果创建 inode 时分配的是相邻的 data block，那我写 bitmap block 的时候，只需要修改一个 bitmap block 的很多 bit 位就好了。
3. 最后是 disk scheduling，首先，向磁盘写入连续的 1000 个 block，比分 1000 次每次写连续位置的 disk block 要快得多（机械硬盘尤其如此），写 log 就是一次向磁盘连续位置写入 block。

> 当我们向文件系统分区写入包含在一个大的transaction中的多个更新时，如果我们能将大量的写请求同时发送到驱动，即使它们位于磁盘的不同位置，我们也使得磁盘可以调度这些写请求，并以特定的顺序执行这些写请求，驱动可以对这些写请求按照轨道号排序，提升写入性能，对固态硬盘来说，这个操作也有一点用处。

## concurrency

除前面异步系统调用提到的 I/O concurrency 之外，还有两种 concurrency：

1. ext3 允许多个系统调用同时执行，这里 xv6 也可以说允许，但 xv6 的并发中，如果 transaction 处于 commiting 状态，即正在提交，那么其他系统调用会直接在 `begin_op` 处 sleep。
> 这里还是不太好理解，ext3 的并行相比 xv6 的优势？是说 ext3 处于 commiting 状态的时候，也能处理新的系统调用嘛？TODO(zwyyy)

2. 可以有多个不同状态的 transaction 同时存在，但是只有一个内存中的 open transaction 可以接收系统调用，但是其他之前的 transaction 可以并行地写磁盘。这里并行存在的不同 transaction 状态包括：
- 一个 open transaction；
- 若干个正在 commit 到 log 的transaction，我们不需要等这些 transaction 结束，若之前的 transaction 还没有 commit 完成，还在写 log，新的系统调用仍然可以在当前 open transaction 中执行；
- 若干个正在从 cache 中向文件系统 block 写数据的 transaction；
- 若干个正在被释放的 transaction；

xv6 中新的系统调用需要等待前一个 transaction 完全完成才能开始。

假设一个 block cache 正在被更新，而这个 block 又在被写入到磁盘，为了保证 transaction 写入到 log 的内容只包含本次 transaction 的更新，那么当它决定结束当前 open transaction，开启一个新的 transaction 时，它会在内存中拷贝所有相关的 block，之后 transaction 的 commit 是基于这些 block 的拷贝进行的，操作系统会使用 copy-on-write 来避免不必要的拷贝，即新 transaction 对这个 block cache 发生写入时，才真正拷贝这个 block cache（可以说之前只是添加了一份引用。

concurrency 之所以能帮助提升性能，正是因为它帮助我们并行地运行系统调用。

