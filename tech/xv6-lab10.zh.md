---
title: "Xv6 Lab10: file system"
date: 2023-08-01T20:19:47+08:00
lastmod: 2023-08-01T20:19:47+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["xv6", "os", "lab", "mit"]
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
## Large files

这个作业需要我们将 xv6 的最大文件大小从 `12 + 256` Bytes 修改为 `11 + 256 + 256 * 256` Bytes。

为了达成这个目标，我们需要使用二级索引块，对 inode 的 `addrs` 字段，首先将 `NDIRECT` 从 $12$ 修改为 $11$，即前 $11$ 个 block 是 direct block，`addrs[NDIRECT]` 对应的块是一个一级索引块，这个块中的每个元素（共 `BSIZE / sizeof(uint)` 个元素）都是一个数据块的编号；而 `addrs[NDIRECT + 1]` 是一个二级索引块，这个块中的每个元素都是一级索引块的编号，由编号找到一级索引块，然后再由以及索引块找到数据块。

这个作业的主要任务就是修改 `bmap` 和 `itrunc` 两个函数。

先修改 `fs.h` 中的 `NDIRECT` 的相关定义：

```c
#define FSMAGIC 0x10203040

#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)

struct dinode {
    short type;              // File type
    short major;             // Major device number (T_DEVICE only)
    short minor;             // Minor device number (T_DEVICE only)
    short nlink;             // Number of links to inode in file system
    uint size;               // Size of file (bytes)
    uint addrs[NDIRECT + 2]; // Data block addresses
};
```

注意 `file.h` 中的 `inode` 结构体也要做相应的修改。

修改 `bmap` 函数，用商来表示一级块的编号，模来表示数据块的编号。

```c
static uint bmap(struct inode *ip, uint bn) {
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

    bn -= NINDIRECT;
    if (bn < NDINDIRECT) {
        if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
            addr = balloc(ip->dev);
            ip->addrs[NDIRECT + 1] = addr;
        }
        struct buf *tmp_buf = bread(ip->dev, addr);
        uint *arr_tmp = (uint *)tmp_buf->data;
        uint lv1_addr;
        if ((lv1_addr = arr_tmp[bn / NINDIRECT]) == 0) {
            lv1_addr = balloc(ip->dev);
            arr_tmp[bn / NINDIRECT] = lv1_addr;
            log_write(tmp_buf);
        }
        brelse(tmp_buf);
        bp = bread(ip->dev, lv1_addr);
        a = (uint *)bp->data;
        if ((addr = a[bn % NINDIRECT]) == 0) {
            a[bn % NINDIRECT] = addr = balloc(ip->dev);
            log_write(bp);
        }
        brelse(bp);
        return addr;
    }

    panic("bmap: out of range");
}
```

修改 `itrunc`，如果了解 inode 的数据在磁盘中分布方式的话，free 起来也很简单，照着原先的 `itrunc` 写就好了。先释放数据块，然后释放这些数据块对应的一级索引块，最后释放这些数据块对应的二级索引块。

```c
void itrunc(struct inode *ip) {
    int i, j;
    struct buf *bp;
    uint *a;

    for (i = 0; i < NDIRECT; i++) {
        if (ip->addrs[i]) {
            bfree(ip->dev, ip->addrs[i]);
            ip->addrs[i] = 0;
        }
    }

    if (ip->addrs[NDIRECT]) {
        bp = bread(ip->dev, ip->addrs[NDIRECT]);
        a = (uint *)bp->data;
        for (j = 0; j < NINDIRECT; j++) {
            if (a[j])
                bfree(ip->dev, a[j]);
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT]);
        ip->addrs[NDIRECT] = 0;
    }

    if (ip->addrs[NDIRECT + 1]) {
        bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
        a = (uint *)bp->data;
        for (int j = 0; j < NINDIRECT; ++j) {
            if (a[j]) {
                struct buf *sub_bp = bread(ip->dev, a[j]);
                uint *addr = (uint *)sub_bp->data;
                for (int k = 0; k < NINDIRECT; ++k) {
                    if (addr[k]) {
                        bfree(ip->dev, addr[k]);
                    }
                }
                brelse(sub_bp);
                bfree(ip->dev, a[j]);
            }
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT + 1]);
        ip->addrs[NDIRECT + 1] = 0;
    }

    ip->size = 0;
    iupdate(ip);
}
```

## Symbolic links
首先要搞清楚硬连接（hard link）与符号连接（soft link or symbolic link）之间的区别。

`int link(const char *target, const char *path)`，我们要为 target 创建一个路径为 path 的硬链接。在硬链接的情况下，两个路径对应着同一个 inode，我们可以先看一下 xv6 中 `sys_link` 的实现，它的核心在于调用 `dirlink` 在 path 的父目录下，创建对应的 directory entry，而这个 entry（即 `dirent`）的 `inum` 和目标文件的 `inum` 是相同的，即当我们通过 `dirlookup` 调用 `iget` 来获取 `inode` 的指针时，两个指针指向的是同一个 inode。

而符号链接 `int symlink(const char *target, const char *path)` 则不一样，`path` 对应的 inode（记为 `ip`） 和 `target` 对应的 inode（记为 tp） 并不是同一个，只不过在 `ip` 的数据块中，存储的是 `tp` 的路径，即字符串 `target`。同时 `ip->type` 是 `T_SYMLINK`。

明确了符号链接与硬链接的区别，这个作业就不难写了（我当时把符号链接跟硬链接搞混了，折腾了很久）。

添加系统调用的步骤不再赘述。直接思考如何实现 `sys_symlink`。思路很简单，利用 `create` 创建一个 inode，**注意这里创建的 inode（即返回的 inode 的指针）是带锁的**！然后利用 `writei` 将数据（即字符串 `target`）写入到 inode，注意这里应该是 kernel page table 下，`writei` 自身会调用 `iupdate`，因此不需要我们手动调用 `iupdate`。

> inode 是直写的，修改了 `inode` 的数据必须调用 `iupdate` 更新到磁盘。

注意整个流程需要用 `begin_op` 和 `end_op` 包裹起来，`iput` 也会调用 `iupdate`，而 `iupdate` 会调用 `log_write`。

```c
uint64 sys_symlink(void) {
    char path[MAXPATH];
    char target[MAXPATH];
    struct inode *ip;
    // printf("symlink\n");
    if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) {
        return -1;
    }
    begin_op();
    // maxpath 为 128，即需要 128 个字节，一个 block 是 1024 个字节
    if ((ip = create(path, T_SYMLINK, 0, 0)) == 0) { // create 返回的是锁定的 ip
        end_op();
        return -1;
    }
    // printf("start_symlink_loop\n");
    if ((writei(ip, 0, (uint64)target, 0, MAXPATH)) < MAXPATH) {
        iunlockput(ip);
        end_op();
        return -1;
    }
    // iupdate(ip);
    iunlockput(ip);
    end_op();
    return 0;
}
```

同时，我们还需要修改 `sys_open`，添加判断 `ip->type` 是否为 `T_SYMLINK` 的处理逻辑，并且需要防止符号链接形成环路，即需要限制循环次数，我这里定为 `#define R_MAX_DEPTH 12`（这里之前没注意提示的内容，被死循环卡了挺久的）。

```c
uint64 sys_open(void) {
    char path[MAXPATH];
    int fd, omode;
    struct file *f;
    struct inode *ip;
    int n;

    if ((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
        return -1;

    begin_op();

    if (omode & O_CREATE) {
        ip = create(path, T_FILE, 0, 0);
        if (ip == 0) {
            end_op();
            return -1;
        }
    } else {
        if ((ip = namei(path)) == 0) {
            end_op();
            return -1;
        }
        ilock(ip);
        if (ip->type == T_DIR && omode != O_RDONLY) {
            iunlockput(ip);
            end_op();
            return -1;
        }
    }

    if (ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)) {
        iunlockput(ip);
        end_op();
        return -1;
    }

    if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
        int read_num;
        int recur_times = 0;
        while (ip->type == T_SYMLINK) {
            if ((read_num = readi(ip, 0, (uint64)path, 0, MAXPATH)) < MAXPATH) {
                iunlockput(ip);
                end_op();
                return -1;
            }
            iunlockput(ip);
            ip = namei(path);
            // printf("symlink, path: %s %d\n", path, 2);
            if (ip == 0 || recur_times++ >= R_MAX_DEPTH) {
                end_op();
                return -1;
            }
            ilock(ip);
        }
    }

    if ((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0) {
        if (f)
            fileclose(f);
        iunlockput(ip);
        end_op();
        return -1;
    }

    if (ip->type == T_DEVICE) {
        f->type = FD_DEVICE;
        f->major = ip->major;
    } else {
        f->type = FD_INODE;
        f->off = 0;
    }
    f->ip = ip;
    f->readable = !(omode & O_WRONLY);
    f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

    if ((omode & O_TRUNC) && ip->type == T_FILE) {
        itrunc(ip);
    }

    iunlock(ip);
    end_op();

    return fd;
}
```