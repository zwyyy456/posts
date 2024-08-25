---
title: "Mit6.s081 Lec15: xv6 的 logging system"
date: 2023-07-25T16:48:39+08:00
lastmod: 2023-07-25T16:48:39+08:00 #更新时间
authors: ["zwyyy456"] #作者
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
## Logging layer

file system 设计的一大重要问题就是 **crash recovery**。这是因为文件系统操作往往涉及向磁盘多次写入，而几次写入之后的 crash 可能导致磁盘上的文件系统处于一个不一致的状态。

> For example, suppose a crash occurs during file truncation (setting the length of a file to zero and freeing its content blocks). Depending on the order of the disk writes, the crash may either **leave an inode with a reference to a content block that is marked free**, or it may **leave an allocated but unreferenced content block**.

前者当系统重启之后,可能导致一个磁盘 block 被两个文件所对应，这是一个很严重的问题。 

xv6 的解决方法是使用 logging（日志）。xv6 的系统调用不会直接向 on-disk file system data structure（例如 buffer 以及磁盘 block？）写入，而是将它所有希望执行的**磁盘写入**的**描述**放在磁盘的某个 log 中，once the system call has logged all of its writes, it writes a special **commit** record to the disk indicating that the log contains a complete operation（这里不知道怎么翻译好了）。此时，系统调用才将 disk write 应用到 on-disk file system data structure。当所有的写入操作完成后，系统调用会清除磁盘上的 log。

如果系统崩溃重启，那么 file-system 代码会在运行进程之前，先检查磁盘上的 log。如果 log 被标记为包含完整的操作，那么 file-system recovery code 会将 log 中的写操作应用到对应的 on-disk file system；否则 file-system recovery code 会直接清除 log，不执行写入操作。

> 如果 file system 恢复时看到 log 被标记为不包含完整操作，那么说明 log 中的写入操作都还没有应用到磁盘上，即磁盘没有发生真正的写入，因此直接清除 log 即可，不需要执行任何写入操作。

log 使得这些写入操作对 crash 来说是原子的，要么都没写入，要么全都写入到磁盘了。

## Log design

log 存在于磁盘的 superblock 中，它由一个 header block 和后面的一系列 updated block copies（更新块的副本，又称 logged blocks） 组成。header block 包含一个扇区号（sector index）的数组，每个 sector index 对应一个 logged block，还包含着 log blocks 的数量。header block 中的计数要么为 $0$，说明 log 中不存在 transaction，要么不为 $0$，说明 log 包含一个完整的已、已提交的 transaction，并包含指定数量的 logged blocks。

xv6 在 transaction 提交时，向 header block 写入数据，在将 logged blocks 复制到 file system 之后，将 header block 的计数置为 $0$。

为了允许 file system 的操作被不同进程并发执行，日志系统（logging system）会将多个系统调用的写操作合并成一个 transaction。因此，单次的 commit 可能涉及多个完整的系统调用的写操作，为了避免一次系统调用被划分成两次 transaction，日志系统只会在没有 file-system 系统调用的时候进行 commit。

一次一起 commit 多个 transaction 的方法被称为 **group commit**，group commit 能通过平摊一次提交的开销到多次操作上来降低所需的磁盘操作数。

xv6 在磁盘上标识出了一段固定大小的空间用来存放 log。因此，系统调用要写入的 blocks 必须能被这块空间容纳。对 `write` 和 `unlink` 系统调用来说，这个限制可能会导致两个问题：

1. 假设 `write` 要写一个大文件，那么可能要写入许多 data blocks、bitmap blocks 和 inode blocks，`unlink` 一个大文件可能要写入许多 bitmap blocks 和 inode blocks。为了解决这一问题，xv6 的 `write` 会将一次大的写入划分成多个更小的适合 log 的写入操作，由于 xv6 只使用一个 bitmap lock，因此 `unlink` 不会造成问题；
2. 除非能确定系统调用的写入能容纳于 log 的剩余空间中，否则系统不会允许系统调用执行；

## Code: logging

系统调用中的 log 的使用一般如下图所示：

![rJgukCV9DnRGfaE](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065958.png)

`begin_op` wait until the logging system is not currently commiting, and until there is enough unreserved log space to hold the writes from this call. `log.outstanding` 会统计预订了 log space 的系统调用的数量，被预订的总空间就是 `log.outstanding` 乘上 `MAXOPBLOCKS`。递增 `log.outstanding` 会预订 log space，并且防止在此系统调用期间发生 commit。 xv6 保守地预计每个系统调用都会写入 `MAXOPBLOCKS` 个不同的 block。

> 事实上 等 `end_op` 完成 `commit()` 之后，`begin_op` 才会被唤醒，此时不仅是 log 被提交，更新也已经被应用到了磁盘。

`log_write` 表现得就像是 `bwrite` 的代理那样。它将 block 的 sector number 记录在内存中，在 disk 的 log 部分给这个 block 预留了一个槽位，并调用了 `bpin(b)` 来保证 buffer b 一定不会被 evict。

> 调用 `log_write` 时，调用者已经完成了对 buffer 的数据的修改；
> file system 中，`bwrite` 只会出现于 `end_op` 中；
> 调用 `bpin(b)` 之后，哪怕 buffer 缓存不足时，buffer b 也不会被换出，它修改了 buffer b 的 `refcnt`，之所以要这样设置，是因为假设 buffer b 中途被换出到磁盘，就违背了“必须 commit 之后再将写入应用到磁盘”的设定了，也就破坏了磁盘写入相对于 crash 的原子性。

```c
void log_write(struct buf *b) {
    int i;

    acquire(&log.lock);
    if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
        panic("too big a transaction");
    if (log.outstanding < 1)
        panic("log_write outside of trans");

    for (i = 0; i < log.lh.n; i++) {
        if (log.lh.block[i] == b->blockno) // log absorption
            break;
    }
    log.lh.block[i] = b->blockno;
    if (i == log.lh.n) { // Add new block to log?
        bpin(b);
        log.lh.n++;
    }
    release(&log.lock);
}
```

在 commit 之前，block 必须要缓存在 buffer 上， 这是因为 buffer 唯一地记录着我们对这个 block 的修改；只有在 commit 之后，才可以将 buffer 写入磁盘上的位置；同一 transaction 的假设要读取这个 block，那么它必须能看到对这个 buffer 的修改。`log_write` 会注意到一个 transaction 中，多次写入同一个 block 的情况，如果是写入同一个 block，那么就不会添加新 block 到 log 中，而是在 log 中为该 block 分配相同的槽位。这种优化通常被称为 **absorption（合并）**，这样的话，从 buffer 写入到 log 的槽位只会发生一次，即使写入 buffer 发生了很多次。

`end_op` 首先会减少未完成的系统调用的计数（说明有系统调用完成了），如果计数被递减到了 $0$，它就会通过调用 `commit()` 来提交当前的 transaction。这个过程分为四个阶段：

1. `write_log` 将每个在 transaction 被修改的 block 的 buffer 复制到 log 对应的槽位中（先复制到了 log 的 buffer 的对应的槽位中，然后将这个 log 的 buffer 写入到磁盘）
2. `write_head` 将 header block 从 buffer 写到磁盘中，而这就是 commit 的关键节点：
    - 当完成 header block 的从 buffer 写入到磁盘后，如果发生了 crash，那么恢复程序会从 log 中执行该次事务的写操作，将数据从 log 的 buffer 写入到磁盘；TODO(zwyyy)，如果 `write_head` 的 `bwrite(buf)` 发生了一半呢？
3. `install_trans` 会从 log 的 buffer 中读取每个 block 的 buffer，然后写入到文件系统（磁盘）；
4. `end_op` 将 log header 的计数重置为 $0$（说明 log 中不存在 transaction），这会在下一个 transaction 要写入 logged blocks 之前完成，这保证了 crash 恢复时，不会将前一个 transaction 的 header 与下一个 transaction 的 logged blocks 混在一起恢复，即保证了一次恢复不会涉及两个 transaction。

`recover_from_log` 会被 `inilog` 调用，而 `initlog` 会被 `fsinit` 调用，这个调用的时间点在第一个用户进程运行之前，即第一个用户进程运行之间，文件系统会进行自检。它会读取 log header，如果 log 包含一次已提交的 transaction，那么它会模拟 `end_op` 的操作。

使用 log 的例子可以参照 `kernel/file.c` 中的 `filewrite`，注意 `writei` 会调用 `write_log`：

```c
begin_op();
ilock(f->ip);
if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
    f->off += r;
iunlock(f->ip);
end_op();
```

## 同一个 transaction 中的并发系统调用

在 xv6 中，不同地系统调用来临时，如果 transaction 没有处于 committing，那么是可以有新的系统调用参与的，它们的执行顺序可以如下图：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070000.png)

这里，可以参与的系统调用数受到了一个限制：`log.lh.n + (log.outstanding + 1) * MAXOPBLOCKS > LOGSIZE)`，其中 `log.lh.n` 在 `log_write` 中是会增加的，由于 log space 在 commit 之前是不会被释放的，`log.outstanding` 只统计了未完成的系统调用数，`end_op` 递减 `outstanding` 不会释放对应的 log space，配合 `log.lh.n` 的限制，才真正避免了 log space 被撑爆。

如上图的调用顺序，在 `begin_op` 中可能不会发生 sleep（假设 log space 够用），因此 `end_op` 唤醒时，可能发现没有进程可以唤醒，但是无伤大雅。

