---
title: "Xv6 Lab9: Locks"
date: 2023-07-29T14:28:58+08:00
lastmod: 2023-07-29T14:28:58+08:00 #更新时间
authors: ["zwyyy456"] #作者
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
## Memory allocator

这一题很简单，主要任务，就是为每个 cpu 维护一个空闲物理内存的链表 `freelist`，xv6 默认使用的结构体 `kmem`，其中包含一个 `freelist` 供所有的 cpu 使用。我们要做的，就是把 `freelist` 修改成 `freelist` 的数组，即 `struct run *freelist[NCPU]`，其中 `NCPU` 是定义于 `kernel/params.h` 的宏，对应 cpu 的个数。 `kmem` 中的 spinlock 是用来保护 `freelist` 的，既然 `freelist` 变成了数组，那么也需要有 `NCPU` 个 spinlock。因此，修改 `kmem` 为如下结构体：

```c
struct {
    struct spinlock lock[NCPU];
    struct run *freelist[NCPU]; // for each cpu, allocate a freelist
} kmem;
```

接着，我们需要修改 `kinit`，让它初始化每个 spinlock。

```c
void kinit() {
    char lockname[6] = {'k', 'm', 'e', 'm', '0', 0};
    for (int i = 0; i < NCPU; ++i) {
        lockname[4] = '0' + i;
        initlock(kmem.lock + i, lockname);
    }
    freerange(end, (void *)PHYSTOP);
}
```

接着，我们需要修改 `freerange`，使得 `freerange` 将范围内所有的空闲内存都分配给运行 `freerange` 的 cpu 的 `freelist`。这里我们定义了一个新的函数 `kfree_cpu` 来实现这一功能（记得在 `defs.h` 中声明），这里要注意一点，调用 `cpuid()` 以及使用它的结果之前，要关闭外部中断防止定时器中断切走线程，之后再开启中断。

```c
void freerange(void *pa_start, void *pa_end) {
    char *p;
    p = (char *)PGROUNDUP((uint64)pa_start);
    push_off();
    int cid = cpuid();
    for (; p + PGSIZE <= (char *)pa_end; p += PGSIZE) {
        kfree_cpu(p, cid);
    }
    pop_off();
}
void kfree_cpu(void *pa, int cid) {
    struct run *r;
    if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP) {
        panic("kfree");
    }
    memset(pa, 1, PGSIZE);
    r = (struct run *)pa;
    r->next = kmem.freelist[cid];
    kmem.freelist[cid] = r;
}
```

`kfree` 也要作对应修改，将空闲内存添加到运行 `kfree` 的 cpu 的 `freelist` 中。

```c
void kfree(void *pa) {
    struct run *r;
    if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);
    r = (struct run *)pa;
    push_off();
    int cid = cpuid();
    // acquire(&kmem.lock);
    r->next = kmem.freelist[cid];
    kmem.freelist[cid] = r;
    pop_off();
    // release(&kmem.lock);
}
```

最后还要修改 `kalloc`，从运行 `kalloc` 的 cpu 的 `freelist` 分配，如果 `freelist` 为空（即 `r == 0`），那么就从其它 cpu 的 `freelist` 处获取，这里存在 race condition，需要上锁，完成操作之后再解锁。

```c
void *kalloc(void) {
    struct run *r;
    // alloc 的时候，需要上锁，考虑到当前 cpu 的 freelist 为空，需要使用锁
    // acquire(&kmem.lock);
    push_off();
    int cid = cpuid();
    r = kmem.freelist[cid];
    if (r) {
        kmem.freelist[cid] = r->next;
    } else {
        // 遍历其他 cpu 的 freelist，找一个有空闲块的 cpu 的 freelist
        for (int i = 1; i < NCPU; ++i) {
            int idx = (cid + i) % NCPU;
            acquire(&kmem.lock[idx]);
            r = kmem.freelist[idx];
            if (r) {
                kmem.freelist[idx] = r->next;
                release(&kmem.lock[idx]);
                break;
            }
            release(&kmem.lock[idx]);
        }
    }
    pop_off();
    // release(&kmem.lock);
    if (r)
        memset((char *)r, 5, PGSIZE); // fill with junk
    return (void *)r;
}
```

## Buffer cache

TODO(zwyyy): clockintr 中，为什么要 `wakeup(&tickslock)`;

这个实验室需要我们重新设计 `bcache` 的数据结构，从单纯的链表变成哈希表，哈希表的 bucket 数量在提示中说明了，可以为 $13$。一开始的时候，我想着不如不使用 `bcache`，直接另外设计一个哈希表，哈希表的每个 bucket 是一个无虚拟头结点的双向链表，但是写好之后发现一直发生死锁，搞得 `bget` 非常复杂，尤其是哈希表的 bucket 中的元素要移动到另一个 bucket 中时。

后来将数据结构改成了如下形式：

```c
struct entry {
    struct buf head; // val
    struct spinlock lock;
};

struct {
    struct spinlock lock;
    struct buf buf[NBUF];

    struct entry table[NBUCKET];

    // Linked list of all buffers, through prev/next.
    // Sorted by how recently the buffer was used.
    // head.next is most recent, head.prev is least.
} bcache;
```

即 `bcache` 有三个成员变量，`lock`、buf 数组 `buf` 和哈希表 `table`，`table` 其实相当于是一个数组，数组中的元素是一个 spinlock 和一个 `struct buf` 链表的虚拟头结点，相比原始的 `bcache`，可以理解为把一个链表分成了 `NBUCKET` 个链表，同时每个链表有一个 spinlock 来保护链表的数据，链表中结点的插入、删除与原始的 `bcache` 类似。

但由于我们不需要通过链表的顺序来实现 lru 策略，因此，插入的时候直接作为对应链表的虚拟头结点的 next 即可，即作为实际的头结点。

在 `bget` 中，最难实现的部分，就是选择一个 victim buf，然后将它重用为当前 block 的 cache。在这里非常容易出现死锁之类的问题，事实上，为了保护这里的 invariant，即一个 block 只能有一份 block cache，并且只会选择 `refcnt == 0` 的 buf 作为 victim buf，需要用到 bcache.lock 和哈希表的 bucket 的 spinlock，而我一开始只使用了哈希表的 bucket 的 spinlock，加上没有虚拟头结点导致链表的插入、删除相对麻烦，因此不是发生 `panic` 就是死锁。

### `binit`

明确了数据结构和思路之后，让我们来一步步实现，首先是 `binit`，它要实现的功能可以说分为两步：

1. 初始化 `bcache`，即 `bcache` 的 spinlock 和哈希表，前面提到，哈希表可以理解为把原来 `bcache` 中的链表分解成了 `NBUCKET` 个链表，对每个链接，让头结点的 `next` 和 `prev` 都指向头结点自身来表示链表为空；
2. 初始化哈希表中的 `NBUCKET` 个链表，以及每个链表的 spinlock。原始的 `bio.c` 中，需要把 `NBUF` 个 buf 插入到 `bcache` 的链表中去，而这里我们需要把 `NBUF` 个 buf 插入到这 `NBUCKET` 个链表中去，根据 buf 在 buf 数组中的索引对 `NBUCKET` 取的模的值，来判断它应该放到哪个链表中去。

```c
void binit(void) {
    initlock(&bcache.lock, "bcache");
    for (int i = 0; i < NBUCKET; ++i) {
        bcache.table[i].head.next = &bcache.table[i].head;
        bcache.table[i].head.prev = &bcache.table[i].head;
        // lockname[6] = 'a' + i;
        initlock(&bcache.table[i].lock, "bucket");
    }
    struct buf *b;
    for (int i = 0; i < NBUF; ++i) {
        int key = i % NBUCKET;
        // 无论 table 中的链表是否为空，都可以这样来处理
        b = bcache.buf + i;
        b->key = key;
        b->next = bcache.table[key].head.next;
        b->prev = &bcache.table[key].head;
        initsleeplock(&b->lock, "buffer");
        bcache.table[key].head.next->prev = b;
        bcache.table[key].head.next = b;
    }
}
```

### `bget` 

`bget` 的实现也可以分成两部分：

1. block cache 存在于 `bcache` 中；
2. block cache 不存在于 `bcache` 中，需要从 `bcache` 中寻找 victim buf，然后重用它。

block cache 存在 `bcache` 的情况下，处理很简单，我们首先由 `(dev + 1) * blockno` 计算出 target block cahce 的 `key`，然后 `key % NBUCKET` 确定该 block cache 位于哪个 bucket，然后对该 bucket 上锁（因为我们需要修改 block cache 的 `b->refcnt`），遍历 bucket 的链表，找到 `dev` 与 `blockno` 都对应的那个 block cache `b`，然后递增 `b->refcnt`，释放 bucket 的链表的锁。然后请求 `b` 的 sleeplock，这里主要是考虑到 `b` 可能要将内容写入到磁盘，需要花费很长时间，同时不希望 `b` 的内容被其他进程看到，更不希望被修改，所以请求 `sleeplock`，最后 `return b`；

`bcache` 中没有这个 `block` 的 cache 的情况下，处理逻辑就非常复杂了，首先我们需要寻找一个 victim buf，victim buf 一定是 `refcnt == 0` 的所有 buf 中，ticks 值最小的那个。并且，我们需要保证，当前进程始终持有 victim 所在的 bucket 的锁。

> 我一开始是这样做的：每次遍历一个 bucket 的链表时持有锁，遍历完就释放，当遍历完所有 bucket 确定下来 victim 之后，需要将 victim 从所在 bucket 中移除，此时先申请bucket 的 spinlock，完成删除之后再释放；然后申请 `key` 对应的 bucket 的 spinlock，将 victim 插入到 `key` 对应的 bucket 中，然后释放 spinlock。

> 这样做的问题在于，找到了 victim 和将 victim 从当前 bucket 中移除时，有一段空隙，当前进程并没有持有 victim 所在的 bucket 的锁，那么此时其他进程就可能修改 victim 的值，例如将 `refcnt` 从 $0$ 设置为了 $1$，那么就会出现两个 disk block 对应同一个 block cache 的错误情况。

因此，必须保证当前进程始终持有 `victim` 所在 bucket 的锁，直到将 victim 从当前 bucket 中移除为止。

我是这样实现了，遍历所有的 bucket 来寻找 victim，先申请当前 bucket 的锁，假设没有满足条件的 victim，那么就释放正在遍历的 bucket 的锁，然后开始遍历下一个 bucket；假设找到了满足条件的 victim，如果旧的 victim 和这个新的 victim 不在同一个 bucket 中，那么释放旧 victim 的 bucket 的锁，更新 victim，否则如果旧 victim 和新 victim 位于同一个 bucket 中，那么只更新 victim，不释放锁。

这样操作后，在寻找 victim 和将 victim 从其原本所在的 bucket 中删除之前，进程会始终持有 victim 所在 bucket 的锁。然后申请 `key` 对应的 bucket 的锁，将 victim 插入到此 bucket 中。

综上， `bget` 的实现如下，从实现可以看出，其实没有 `bcache.lock` 的参与就能完成。

```c
static struct buf *bget(uint dev, uint blockno) {
    struct buf *b;
    int i;
    int key = ((dev + 1) * blockno) % NBUCKET;
    // acquire(&bcache.lock);
    acquire(&bcache.table[key].lock);
    for (b = bcache.table[key].head.next; b != &bcache.table[key].head; b = b->next) {
        if (b->dev == dev && b->blockno == blockno) {
            ++b->refcnt;
            release(&bcache.table[key].lock);
            // release(&bcache.lock);
            acquiresleep(&b->lock);
            return b;
        }
    }
    release(&bcache.table[key].lock);
    // Not cached.
    uint stmp = kUintMax;
    int cur_key = 0;
    struct buf *victim;
    // acquire(&bcache.lock);
    // bcache.table[i].lock 必须交错 release，即确保 victim 被更换之前，victim 对应的锁不能被释放
    // 不如直接扫描 bcache 中的所有 buf
    for (i = 0; i < NBUCKET; ++i) {
        acquire(&bcache.table[i].lock);
        int find_flag = 0;
        for (b = bcache.table[i].head.next; b != &bcache.table[i].head; b = b->next) {
            if (b->refcnt == 0 && b->stamp < stmp) {
                stmp = b->stamp;
                if (cur_key != i && holding(&bcache.table[cur_key].lock)) {
                    release(&bcache.table[cur_key].lock);
                }
                victim = b;
                cur_key = i;
                find_flag = 1;
            }
        }
        if (!find_flag) {
            release(&bcache.table[i].lock);
        }
    }
    if (stmp != kUintMax && victim != 0) {
        // acquire(&bcache.table[cur_key].lock);
        victim->dev = dev;
        victim->blockno = blockno;
        victim->valid = 0;
        victim->refcnt = 1;
        victim->key = key;
        // 可能需要移动当前 buf 到其他 bucket
        victim->next->prev = victim->prev;
        victim->prev->next = victim->next; // 从当前链表删除
        release(&bcache.table[cur_key].lock);

        // 插入到新的链表
        acquire(&bcache.table[key].lock);
        victim->next = bcache.table[key].head.next;
        victim->prev = &bcache.table[key].head;
        bcache.table[key].head.next->prev = victim;
        bcache.table[key].head.next = victim;
        release(&bcache.table[key].lock);
        // release(&bcache.lock);
        acquiresleep(&victim->lock);
        return victim;
    }

    panic("bget: no buffers");
}
```

注意，我们需要修改 `struct buf`，为其添加 `uint stamp` 和 `int key` 两个字段，前者一看便知，后者则是说，假如此 `buf` 在内存中存在 cache block，那么这个 cache block 位于索引为 `key` 的 bucket 中。

### `brelse`、`bpin`、`bunpin`

最麻烦的函数其实就是 `bget` 了，剩下这三个都很好处理。`brelse` 处理 lru 不再需要执行链表的插入删除，只需要递减 `buf->refcnt` 并将当前 `ticks` 赋值给 `b->stamp` 即可，这里要申请 `buf` 所在 bucket 的锁，以及 `tickslock`。

```c
void brelse(struct buf *b) {
    if (!holdingsleep(&b->lock)) {
        // printf("panic, brelese\n");
        panic("brelse");
    }
    releasesleep(&b->lock);
    acquire(&bcache.table[b->key].lock);
    b->refcnt--;

    acquire(&tickslock);
    b->stamp = ticks;
    release(&tickslock);

    release(&bcache.table[b->key].lock);
}
```

对于 `bpin` 和 `bunpin`，只需要修改持有和释放的锁即可，变成持有 `buf` 对应的 bucket 的锁。

```c
void bpin(struct buf *b) {
    acquire(&bcache.table[b->key].lock);
    b->refcnt++;
    release(&bcache.table[b->key].lock);
}

void bunpin(struct buf *b) {
    acquire(&bcache.table[b->key].lock);
    b->refcnt--;
    release(&bcache.table[b->key].lock);
}
```

## 总结

第一个实验很简单，就不再重复了。第二个实验属实非常酸爽，各种死锁、panic，针对这种死锁的定位和调试，不知道各位有没有什么好办法。另外，个人感觉，在涉及锁的并行编程中，一个精心设计的数据结构能够让你在实现的时候少很多麻烦，例如这里使用带虚拟头结点的循环双向链表来作为 bucket 中的数据结构，而不是简单的双向链表（带虚拟头结点的虚拟尾结点的双向链表应该也可以），因为简单双向链表，边界条件处理起来很麻烦，本身并行编程为了防止 race condition 和 deadlock 就已经很麻烦了，两个麻烦叠加起来绝对是 $1 + 1 > 2$。

另外一点，则是要考虑清楚，我使用锁，到底要保护一个什么样的 invariant，在这题中，我要保证 victim 始终是一个合法的 victim，它的 `refcnt` 一定不能中途被其他进程修改。


