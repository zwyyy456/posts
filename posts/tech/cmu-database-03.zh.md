---
title: "Cmu Database 03"
date: 2023-08-23T10:40:13+08:00
lastmod: 2023-08-23T10:40:13+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["database"]
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

## 为什么 dbms 不会使用 os 中的 mmap

###

transaction safety

### I/O stalls

DBMS don't know which pages are in memory. The OS will stall the thread on page fault.

### Error handling

any access can cause a SIGBUS

### Perfomance Issues

OS data structure contention. TLB shootdowns.

## Database pages

A page is a fixed-size block of data.

Most system do not mix page types.

Some systems require a page to be self-contained. 所有如何解释这个页面的信息都必须存在于这个页面中，类比一下 elf。

在 DBMS 中存在三种类型的 page：
- hardware page，通常 4KB；（也称为 Block 或者 Sector）
- OS Page，通常是 4KB;（也称为 Memory page）
- Database Page，(512B-16KB)

hardware page 的写入能保证是原子的，写入一个 hardware page，要么全部成功，要么全部不成功。

## HEAP FILE

### PAGE DIRECTORY

### PAGE Header

header 包含了 page size、checksum、DBMS version、transaction visibility、compression information，某些系统要求 "pages to be self-contained"。

### sloted pages

slot array 位于 page 的开头（header 之后），tuple array 会从 page 的末尾向开头增长。slot array 中可以存储 tuple 在 page 中的 offset 以及这个 tuple 的大小。

### Record IDS

每个 tuple 都会被赋予一个唯一的 record identifier，一般来说由 `page_id` + `offset / slot` 组成。

在 postgresql 中，被称为 `CTID`，sqlite 和 orcacle 中被称为 `rowid`。


## Log-Structured Storage

区分于 page-oriented storage

这里的 log 是持久化的，log 保存的是 database 发生的修改（`put` 和 `delete`，类似于哈希表键值对的 `put` 和 `delete`，key 已经存在时，`put` 就是修改），每个 log record 都包含着 tuple 的唯一标识，即每个 log 修改一个 tuple。

通过找到 tuple 对应的最新的 log，我们就能得知 tuple 现在的值。

log-structured page 有利于减少 random disk I/O，这是因为我们可以将写入序列化。

在日志式的存储中，会有一个索引指向 log 中的特定位置（哈希表或者 B+ tree？），同时，DBMS 可能会周期性地压缩 log，即对每个 tuple，只保留最新的那个 log。

在日志式存储结构中，对 log 进行压缩之后，可能会根据 id 对 log 进行排序，这样排序处理之后的 log 被称为 **Sorted String Tables**，因此我们可以很快找到 tuple 对应的 log。

日志式存储结构的问题在于，在压缩 log 的时候，存在着 **write amplification** 的情况，一条日志（假设没有更新的替换它），在压缩的过程中，可能会反复地被写入磁盘。

> 将压缩好的日志写入磁盘时，又将日志写入到了磁盘一次。

## Data representation

### Variable-Length data

大多数的 DBMS 都不允许 tuple 超过单个 page 的大小，对于这样的 tuple（即大小超过 page size 的 tuple），DBMS 可能会将它存储在 **overflow page** 中，普通 page 中，该 tuple 的数据被保存为该 overflow page 的引用。overflow page 可以保存其他 overflow page 的指针，直到数据可以被保存下来了。（类似 xv6 中的 indirect block）。

某些系统可能会将很大的数据存储在 external file 中，DBMS 中保存的是该 file 的 link。这样做的缺陷是无法保证 external file 的原子性。

### System catalogs

DBMS 会在 page 中保存 internal catalog 来标识数据库的 meta-data，例如数据库有哪些 table，哪些 columns，数据的顺序。





