---
title: "MIT 6.S081 Lec14: File system"
date: 2023-07-22T19:51:29+08:00
lastmod: 2023-07-22T19:51:29+08:00 #更新时间
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
## Overview

文件系统的设计目标就是组织和存储数据，文件系统一个比较重要功能是**持久化**，即重启之后，数据不会丢失。xv6 通过把数据存储在 virtio disk 上来实现**持久化**。

文件系统设计的几大挑战：

- The file system needs on-disk data structures to represent the tree of named directories and files, to record the identities of the blocks that hold each file’s content, and to record which areas of the disk are free.
- 由于文件系统需要实现**持久化**，因此必须要实现 **crash recovery**，即如果发生意外的 crash（例如断电），文件系统在计算机重启之后要能依旧正常工作；
- 可能有多个进程同时操作文件系统；
- 由于访问磁盘比访问内存要慢得多得多，因此文件系统需要能支持将部分 popluar 的 blocks 缓存在内存中；

xv6 的文件系统可以说组织为七层，如下图所示：

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070029.png)

disk layer 负责读写 virtio hard drive 上的 blocks，buffer cache layer 是 blocks 的 cache，并且保证同一时间，只有一个内核进程可以修改存储在特定块上的数据；logging layer 将对几个特定 block 的更新打包为一次 transaction（就是数据库常说的事务？），从而确保这些 blocks 都是被原子化地更新，即要么一次都更新，要么一次都不更新；inode layer 则是用来表示单独的文件，每个文件都是以具有不重复的 index 的 inode 和保存了这个文件的数据的一些 blocks 来表示；而在 directory layer，每个 directory 都是一种特殊的 inode，包含一系列 direcotry entry，directory entry 则是包含了文件名和 index（对应 indode layer 所说的 index）；pathname layer 提供了层级化的路径名，利用递归查找来解析它们；

disk hardware 一般将磁盘划分为 $512$ bytes 的 block 的序列（这个 $512$ bytes 的 block 又称为 sector（扇区））。xv6 使用的 block 的大小一般是 sector 大小的整数倍。

![](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070030.png)

xv6 将磁盘划分为几个功能部分，如下图所示：

![4wINxhLbqZXPMBE](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-070035.png)

## Buffer cache layer

buffer cache 的实现代码在 `kernel/bio.c`， 它主要有两个任务：

1. 同步进程对磁盘的访问，内存中只有一份磁盘的 block 的拷贝，并且同一时间内只有一个进程可以使用这份拷贝（同时读呢，是否可行？）；
2. 缓存 popluar blocks 到内存中，从而可以直接从内存访问磁盘的 popular blocks，避免再从速度极其慢的磁盘中读取；

> 这里记 `struct buf` 的一个实例为 buffer

buffer cache 的主要接口是 `bread` 和 `bwrite`， `bread` 返回一个存储于内存中的 buffer（会被加锁），我们可以读取或者修改这个 buffer，`bwrite` 负责将一个修改过的 buffer 写回到磁盘中合适的 block 去，当完成 `bwrite` 时，内核线程需要调用 `brelse` 来释放 buffer 的锁（是一个 sleep-lock，可以理解为与自旋锁相对？）

> `bread` 会返回带锁的 `buf`，因为 `bread` 调用 `bget`，而 `bget` 会加锁，加的是 sleep-lock，相当于 pthread 中的互斥锁（与自旋锁 spin-lock 相对）；
> 使用 sleep-lock 是因为磁盘相关的操作，可能会消耗比较长的时间；

那些 poplular 的 blocks 其实被缓存在一个 buffer 的双向链表里面（这个双向链表是**首尾相接**的），双向链表的长度是有限的，这就意味着需要有缓存淘汰机制，xv6 中，这里使用了 LRU，`head->prev` 应该是 least recently used，而 `head->next` 则是 most recently。这里的头结点 head 应该是虚拟头结点？TODO(zwyyy)

## Code: Buffer cache

buffer cache 实际上是 buffer 的双向链表，`kernel/main.c` 的 `main` 函数调用 `binit`，使用静态数组 `buf` 中的 `NBUF` 个 `struct buf` 来初始化这个链表，数组只用来初始化，之后访问 buffer 都是通过双向链表都是通过 `bcache.head`，也就是说**这个数组只用来初始化链表！**

buffer 中含有两个状态字段，`valid` 说明这个 buffer 含有磁盘的 block 的拷贝，因此是有效的，`disk` 字段为 $1$ 说明 buffer 已经写回了 disk。

`bread` 可以读取指定 block 对应的 buffer，如果返回的 buffer 的 valid 字段为 $0$，说明 get 是利用 lru 机制选择了一个 victim buffer，因此我们需要调用 `virtio_disk_rw(b, 0)` 将磁盘的对应 block 读入这个 buffer。

`bget` 遍历 buffer 链表寻找与 `dev` 和 `blockno` 对应的 buffer，并为这个 buffer 申请 sleep-lock，返回未解锁的 buffer；如果找不到对应的 buffer，`bget` 会从 head 开始，反向遍历双向链表，找一个 `refcnt` 为 $0$ 的 buffer，将这个 buffer 的 valid 字段标记为 $0$，即说明我们要重新从磁盘对应的 sector 读取数据到 buffer。

`bget` 持有 `bache.lock` 保证了“寻找对应的 buffer 以及如果不存在，为 block 分配对应的 buffer” 这个过程是原子的。

`b->refcnt = 1` 保证了这个 buffer 不会被其他进程拿来作为 victim buffer 去存放其他 block 的数据，因此可以先释放 `bcache.lock` 再申请 `b->lock`。

>  The sleep-lock protects reads and writes of the block’s buffered content, while the bcache.lock protects information about which blocks are cached.

如果 `bread` 的调用者通过 `bread` 拿到了这个 buffer，那么通过 `b->lock`，它享有了对这个 buffer 的独占权，如果它修改了这个 buffer 的内容，那么再释放 `b->lock` 之前，它必须调用 `bwrite` 将 buffer 的被修改的内容写回到 disk。

调用者完成对 buffer 的处理之后，必须调用 `brelse` 将 `b->lock` 释放。



## Code: Block allocator

文件和目录的内容都是存放在 disk blocks 里的，我们必须从未分配的那些 blocks pool 中，为这些数据分配对应的块（就像为程序分配物理内存）。

xv6 的 block allocator 在磁盘上维护了一个 free bitmap，每个 block 对应一个 bit。bit 为 $0$ 表示这个 bit 对应的 block 是空闲的，bit 为 $1$ 表示这个 block 正在被使用。程序 `mkfs` 将这些 bits 与 boot sector、superblock、log blocks、inode blocks、bitmap blocks 对应起来。

`balloc` 的实现代码如下：

```c
static uint balloc(uint dev) {
    int b, bi, m;
    struct buf *bp;

    bp = 0;
    for (b = 0; b < sb.size; b += BPB) {
        bp = bread(dev, BBLOCK(b, sb));
        for (bi = 0; bi < BPB && b + bi < sb.size; bi++) {
            m = 1 << (bi % 8);
            if ((bp->data[bi / 8] & m) == 0) { // Is block free?
                bp->data[bi / 8] |= m;         // Mark block in use.
                log_write(bp);
                brelse(bp);
                bzero(dev, b + bi);
                return b + bi;
            }
        }
        brelse(bp);
    }
    panic("balloc: out of blocks");
}
```

`balloc` 遍历所有的 block，`sb.size` 表示的就是当前文件系统的总的 block 的数量，`balloc` 首先寻找当前 block 对应的 bitmap block，一个 block 占据 $1024$ 个字节，因此，一个 bitmap block 可以表示 $8\times 1024$ 个 block 是否被占用，即 `BPB` 个 block，因此外层循环每次递增 `BPB`，读取这 `BPB` 个 block 对应哪个 bitmap block，然后在内层循环检查每个 block 是否被使用，如果未被使用，将其标记为正在被使用，然后返回。

> 由于 buffer cache 保证了 bitmap block 的 buffer 同一时间只能有一个进程使用，因此避免了 race condtion。

`bfree` 则会直接清除要 free 的 block 在 bitmap block 中的对应 bit。`bread` 和 `brelse` 隐含的独占使用避免了争用的情况。

`balloc` 和 `bfree` 只能在 `transaction` 内部被调用，后面会提到的多数函数也是这样。

## Inode layer

术语 **inode** 可能包含两种含义：

- 一种包含“文件的大小、以及该文件占据的 data blocks 的索引的数组”的 on-disk data structure；
- 指上述的 inode 在内存中的副本以及内核所需要的额外信息；

on-disk inodes 被打包到了磁盘中的 **inode blocks** 区域，这个区域是连续的，每个 inode 都是相同的固定大小，因此我们可以通过 inode 的索引 $n$ 来找到磁盘上的这个 inode。

> inode 的索引被称为 inode number，又称 i-number。
> 同一个文件可能有多个文件名（通过 `link` 系统调用实现）

内核会将有指针指向的 inode 保存在内存中的 `itable` 中，如果指向该 inode 的指针数量降低为 $0$，那么内核会从内存中丢弃该 inode。`itable` 的 `ref` 字段统计了指向该 `inode` 的指针数量。

> 指向 inode 的指针可能是来自于文件描述符、当前工作目录、以及 `exec` 等内核的 transient code。

在 xv6 的 inode 相关的代码中，锁或者类似锁的机制一共有四种。

1. `itable.lock` 保证 inode 在 inode table 中最多只会缓存一次，并且保证 `ref` 一定是指向该 inode 的内存中的指针数量；
2. 每个 in-memory inode 都有一个 `lock` 字段（sleep-lock），确保以独占的方式访问 inode 的字段以及该 inode 对应的 content blocks；
3. 如果 inode 的 `ref` 大于 $0$，系统会在 `itable` 中维护 inode 的缓存，并且不会重用该 table entry 为其他 inode 的缓存；
4. `nlink` 大于 $0$ 时，这个 inode 不会被释放。在 on-disk 的 `dinode` 和 `inode` 中都有该字段。

在调用 `iput` 之前，`iget` 返回的指向 inode 的指针始终有效并始终指向该 inode，`iget` 是非独占的（不同于`bread`），即同时可能有多个指针指向该 inode。

> `iget` 的机制既可以让指针长期指向该 inode，还可以避免死锁的情况下防止 race condition（因为都是只读？）

in-memory inode 的主要功能是实现不同进程对 inode 访问的同步，cache 只是次要的，因为有 buffer 的存在。in-memory inode 缓存是直写的，修改了指向该 inode 的缓存，就必须马上调用 `iupdate` 写回磁盘。

> `iupdate` 的直写策略不会破坏 crash 的一致性吗？TODO(zwyyy)
> 似乎还是修改的 inode 对应的 data block 的 buffer，然后调用 `log_write` 向 log 中做标记

## Code: Inode content

on-disk inode 定义于 `kernel/fs.h` 的结构体 `dinode` 中：
- `dinode` 的 `type` 字段标识出该 inode 是普通文件、目录还是设备（特殊文件）。`type` 为 $0$ 表示此 `inode` 是空闲的。
- `nlink` 字段表示有多少文件名指向当前的 inode，用于确定何时释放这个 inode 对应的 data block。
- `size` 字段说明了文件中的内容占据的字节数。
- `addrs` 字段是一个长度为 `NDIRECT + 1`（即 $13$）的数组，前 $12$ 个元素记录了 data block 的编号，指向构成文件的前 $12$ 个 block，最后一个元素记录了一个特殊的 block 的编号，这个 block 里面存储的是 $256$ 个 data block 的编号，每个编号指向构成文件的后最多 $256$ 个 data block 的其中之一，每个编号占据 $4$ 个字节；

> 一个文件最多占 $(12 + 256) * 1024$ 字节大小；

`static uint bmap(struct inode *ip, uint bn)` 返回 inode `ip` 的第 `bn` 个 block 在磁盘中的 block 编号，如果磁盘中的 block 不存在，那么 `bmap` 会调用 `balloc` 来分配。（`ip->addrs[i]` 或者 `ip->addrs[NDIRECT]` 对应的 entry 的值是 $0$，说明这个 block 没有被分配）

```c
static uint
bmap(struct inode *ip, uint bn) {
    uint addr, *a;
    struct buf *bp;
    if (bn < NDIRECT) {
        if ((addr = ip->addrs[bn]) == 0)
            ip->addrs[bn] = addr = balloc(ip->dev);
        return addr;
    }
    bn -= NDIRECT;
    if (bn < NINDIRECT) {
        // Load indirect block, allocating if necessary.
        if ((addr = ip->addrs[NDIRECT]) == 0)
            ip->addrs[NDIRECT] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a = (uint *)bp->data;
        if ((addr = a[bn]) == 0) {
            a[bn] = addr = balloc(ip->dev);
            log_write(bp);
        }
        brelse(bp);
        return addr;
    }
    panic("bmap: out of range");
}
```

### `writei` 与 `readi`

`readi` 首先会确保文件的 offset 以及 offset 加上要读取的字节数不会超过文件的 size，否则返回 error（`return 0;`），此时返回的读取的字节数小于预期要读取的字节数；没有异常，则住循环遍历文件的所有 block，从 buffer 中把数据复制到 `dst` 中。

`writei` 与 `readi` 类似，首先检查 offset 小于或等于文件的 size，同时保证写入文件不会导致文件超过最大文件大小的限制。

`stati` 将 inode 的 metadata 到 `stat` 结构体中，这是用户程序通过 `stat` 系统调用所需要获取的。



## Code：Inodes

xv6 调用 `ialloc` 来分配新的 inode，它遍历所有的 inode 来找到一个空闲的 inode，找到之后将新的 type 写入到这个 inode 的 buffer 中，然后调用 `log_write`，最后调用 `iget` 返回一个内存中 inode（即将这个 inode 写入到 inode cache）中。

`iget` 查找 `itable` 来找到具有给定 `dev` 和 `inode` 的 active entry，如果找到了，就返回指向该 inode 的指针；如果不存在，它不会从磁盘中读，只是从 `itable` 中回收一个 entry 并返回该 entry，**调用 `ilock` 的时候，才真正从磁盘中读取数据到 inode**。

> atcive 是指该 inode 有指针指向，即 `ref > 0`。
> `ilock` 对 inode 上锁，`iunlock` 解锁；可能有多个进程持有指向该 inode 的指针，但是只有一个进程能对该 inode 上锁。

```c
static struct inode *
iget(uint dev, uint inum) {
    struct inode *ip, *empty;
    acquire(&itable.lock);
    // Is the inode already in the table?
    empty = 0;
    for (ip = &itable.inode[0]; ip < &itable.inode[NINODE]; ip++) {
        if (ip->ref > 0 && ip->dev == dev && ip->inum == inum) {
            ip->ref++;
            release(&itable.lock);
            return ip;
        }
        if (empty == 0 && ip->ref == 0) // Remember empty slot.
            empty = ip;
    }
    // Recycle an inode entry.
    if (empty == 0)
        panic("iget: no inodes");
    ip = empty;
    ip->dev = dev;
    ip->inum = inum;
    ip->ref = 1;
    ip->valid = 0;
    release(&itable.lock);

    return ip;
}

void ilock(struct inode *ip) {
    struct buf *bp;
    struct dinode *dip;
    if (ip == 0 || ip->ref < 1)
        panic("ilock");
    acquiresleep(&ip->lock);
    if (ip->valid == 0) {
        bp = bread(ip->dev, IBLOCK(ip->inum, sb));
        dip = (struct dinode *)bp->data + ip->inum % IPB;
        ip->type = dip->type;
        ip->major = dip->major;
        ip->minor = dip->minor;
        ip->nlink = dip->nlink;
        ip->size = dip->size;
        memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
        brelse(bp);
        ip->valid = 1;
        if (ip->type == 0)
            panic("ilock: no type");
    }
}
```

`iput` 会递减 `ref` 并释放一个指向该 inode 的指针，如果 `ref` 变为 $0$，那么 `itable` 中原本属于该 inode 的位置就能用来存放另外的 inode 缓存。如果 `iput` 发现即没有指针指向该 inode，又没有文件名或者目录指向该 inode，那么该 inode 和他的 data blocks 会被释放，`iput` 会调用 `itrunc` 将文件大小缩短至 $0$，释放对应的 data blocks，然后 `iput` 将 inode type 置为 $0$，将该 inode 写回磁盘。

在 `inode->ref` 是 $1$ 的时候，系统调用无法获取这个 in-memory inode 的指针，这个 `inode->ref` 为 $1$ 的计数是调用 `iput` 的进程持有的。

`iput` 会向磁盘写入，这意味着任何 file system call 都可能向磁盘写入（因为系统调用可能是最后一个引用该 inode 的系统调用），哪怕它看起来只是读取磁盘，因此，所有 file system call 都应该打包成 transaction 来处理。

## Code：directory layer

目录其内部实现其实很像一个文件，只不过它的 inode `type` 是 `T_DIR`，这个 inode 的 data blocks 里面的数据是一系列的 directory entry，每个 entry 都是一个 `dirent` 结构体。inode 为 $0$ 的 directory entry 是空闲的。

```c
struct dirent {
    ushort inum;
    char name[DIRSIZ];
};
```

`dirlookup` 会在给定目录 `dp` 中，查找给定“路径名”对应的 target inode，并将 `poff` 设定为 target inode 在 `dp` 中的 offset，然后返回 target inode 的指针（通过 `iget`）。

`dirlookup` 是 `iget` 不能返回上锁的 inode 的原因之一，`dirlookup` 的调用者会对 `dp` 上锁，假设要查询的目录就是 `.`，即当前目录，那么调用者对 `dp` 上锁，`iget` 也会对 `dp` 上锁，就发生了死锁，因此 `iget` 不能返回上锁的 inode（`iget` 只对 `itable` 上锁，返回时会解锁）。`dirlookup` 的调用者可以先解锁 `dp`，再锁定 `ip`。

```c
struct inode *dirlookup(struct inode *dp, char *name, uint *poff) {
    uint off, inum;
    struct dirent de;
    if (dp->type != T_DIR)
        panic("dirlookup not DIR");
    for (off = 0; off < dp->size; off += sizeof(de)) {
        if (readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
            panic("dirlookup read");
        if (de.inum == 0)
            continue;
        if (namecmp(name, de.name) == 0) {
            // entry matches path element
            if (poff)
                *poff = off;
            inum = de.inum;
            return iget(dp->dev, inum);
        }
    }
    return 0;
}
```

`dirlink` 会向给定目录的 inode 写入一个具有指定路径名的新 directory entry，如果这个名字已经存在了，那么 `dirlink` 会 return error。之后，`dirlink` 在主循环中寻找未分配的 directory entry，如果找到了，就向这个未分配的 entry 对应的 offset 来进行写入。

```c
// Write a new directory entry (name, inum) into the directory dp.
int dirlink(struct inode *dp, char *name, uint inum) {
    int off;
    struct dirent de;
    struct inode *ip;
    // Check that name is not present.
    if ((ip = dirlookup(dp, name, 0)) != 0) {
        iput(ip);
        return -1;
    }
    // Look for an empty dirent.
    for (off = 0; off < dp->size; off += sizeof(de)) {
        if (readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
            panic("dirlink read");
        if (de.inum == 0)
            break;
    }
    strncpy(de.name, name, DIRSIZ);
    de.inum = inum;
    if (writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
        panic("dirlink");
    return 0;
}
```

## Code: Path names

路径名的查找涉及一系列的 `dirlookup`，例如查找 `/x/y/z`，就要先查到根目录下的 x，再查找 x 目录下的 `y/z`，再查找 y 目录下的 z。`namei` 直接调用 `namex` 来查找路径名对应的 inode；`nameiparent` 则是返回要查询的路径的父文件夹对应的 inode 并将最后一个元素复制到 name（例如查询 `y/x/z`，返回 x 对应的 inode，`name` 设置为 z）。

> 注意 C 中空字符串是 `"\0"`，因此 `skipelem("a", name)` 的返回值不是 $0$！

`namex` 首先检查路径是相对路径还是绝对路径，如果路径以 '/' 开头，那么从 root inode 开始查找，否则以当前目录对应的 inode 开始查找。然后使用 `skipelem` 来获取路径的每一级的目录名，例如 `skipelem("x/y/z", name) = y/z`，而 `name = x`，每次循环中，都会更新 `ip` 为 `dirlookup(ip, name, 0)` 返回的 inode 的指针。

> 之所以要用 `ilock` 锁定 inode，是因为运行 `ilock` 前，不能确保一定从磁盘读取了 `ip->type`，有可能这个 inode 只是被分配了，数据还没有从磁盘或者 buffer 更新。

```c
static struct inode *namex(char *path, int nameiparent, char *name) {
    struct inode *ip, *next;
    if (*path == '/')
        ip = iget(ROOTDEV, ROOTINO);
    else
        ip = idup(myproc()->cwd);
    while ((path = skipelem(path, name)) != 0) {
        ilock(ip);
        if (ip->type != T_DIR) {
            iunlockput(ip);
            return 0;
        }
        if (nameiparent && *path == '\0') {
            // Stop one level early.
            iunlock(ip);
            return ip;
        }
        if ((next = dirlookup(ip, name, 0)) == 0) {
            iunlockput(ip);
            return 0;
        }
        iunlockput(ip);
        ip = next;
    }
    if (nameiparent) {
        iput(ip);
        return 0;
    }
    return ip;
}
```

`namex` 可能需要花费很长的时间，xv6 通过精心设计，确保当 `namex` 的一次调用被磁盘 I/O 阻塞时，另一个查找不同路径的内核进程可以并发执行。`namex` 对路径中的每层目录会单独上锁，所以查找不同目录的进程可以并行执行。

在 `namex` 执行 `dirlookup` 的时候，`namex` 持有当前目录 inode 的锁，而 `dirlookup` 返回的时候会调用 `iget`，`iget` 会递增当前 inode 的引用计数（*接下来是个人理解*），只有在 `dirlookup` 返回了 `next` 之后，才会调用 `iunlockput` 来解除对 `ip` 的锁并且递减 `ip` 的引用计数，此时已经可以释放并删除 `ip` 对应的 inode 了，而即使此刻我们 unlink `next` 对应的 inode，xv6 也不会删除这个 inode，因此 `next` 的引用计数被 `iget` 递增了。

这样就避免了，在执行 `dirlookup` 的时候，正在查找的 inode（与 target inode 相对），已经被另一个内核进程删除且 inode 的 block 已经被用作他途了。

查找 `.` 时，返回的 `next` 与当前 `ip` 相等，而 `iget` 不会对当前 inode（即 `ip` 上锁），`namex` 在获得下一个目录的锁之前会释放锁，于是就避免了死锁。

## File descriptor layer

每个进程会维护自己打开的文件描述符的 table，每个文件由 `struct file` 表示（位于 `kernel/file.h`），文件的类型要么是 pipe，要么是 inode（需要配合 offset）。进程每次调用 `open` 都会创建一个新的文件描述符，和一个新的 `struct file`，如果多个进程独立地打开同一个文件，那么它们的文件描述符和 `struct file` 是不同的，但是 `file` 中的 inode 指针是相同的，而 offset 也自然会因为进程对文件的操作而不同。而相同的 `struct file` 也可能出现在不同的进程中，这可能通过调用 `dup` 或者 `fork` 来实现（此时 `file` 中的 `ref` 字段会递增）。

系统中所有打开的文件都保存在一个全局的 `ftable` 中（定义于 `kernel/file.c`），其中还有 `filealloc`、`filedup`、`fileclose`，这三者调用的时候都会请求 `ftable.lock`。这三者看一下源码，很容易就能理解它们在干什么。调用 `fileclose` 时，如果发现 file 的 `ref` 被递减到了 $0$，那么 `fileclose` 会调用 `iput` 来递减 inode 的 `ref`，这个过程被打包成了 transaction。

> 注意，`iput` 会调用 `iupdate`，而 `iupdate` 在完成对 inode 的更新后，会调用 `log_write`。

`filestat` 只允许对 inode 类型的 file 操作，调用 `stati` 来实现。`fileread` 和 `filewrite` 会检查读或者写操作是否在 `sys_open` 时被允许，然后调用 pipe 或者 inode 的对应实现。对 inode 的操作会调用 `ilock` 上锁，因此多个进程写不可能同时写同一个文件，只能依次写，但由于多个进程的 offset 不一致，最终表现出来文件的内容可能还是交错的。

## Code: System calls

通过使用 lower layers 提供的函数，大多数 file-system call 的实现都不难，但是我们还是得关注几个系统调用。

### `sys_link` 与 `sys_unlink`

这两个函数都是 transaction 的很好的例子。`sys_link` 通过 `argaddr` 获取 `old` 和 `new` 两个路径名参数，先对 `old` 作检查（包括是否有对应 inode，inode 是否是目录类型），然后递增该 inode 的 `nlink`，然后查找 `new` 目录的父目录，利用 `dirlink` 在这个父目录下创建一个 directory entry，指向 inode，实现 `new` 与 inode 的绑定。

> 注意其中的 `begin_op` 和 `end_op`。

这个过程涉及对多个 disk blocks 的更新，但是借助 transaction，我们不需要关注更新的顺序，因为我们要么全都成功更新，要么全没有更新。

`sys_link` 的功能其实就是为已存在的 inode 创建一个新的别名（可能位于另外的目录下），

函数 `create` 为一个新的名字创建一个新的 inode，它是 `sys_open`、`sys_mkdir`、`sys_mkdev` 的泛化，后三者的实现都需要调用 `create`。`create` 的实现根据源码来看，不难理解。值的注意的是，`create` 会同时持有 `ip` 和 `ip` 的父目录对应的 inode `dp` 的锁，但这不会造成死锁，因为 `ip` 是新分配的，不会有进程持有这个 `ip` 的锁然后来试图持有 `dp` 的锁。

`sys_open` 根据传入的 flag 的不同，会有不同的行为。flag 是 `O_CREATE` 时，会调用 `create` 来创建，它会返回上锁的 `ip`，而 `namei` 则返回无锁的 `ip`（因此必须要 `sys_open` 自身来锁定）。`sys_open` 会创建一个 file，分配一个 `fd` 与这个 `file` 对应，返回该 `fd`。

> 其他进程无法访问部分初始化的 file，因为它仅位于当前进程的表中。