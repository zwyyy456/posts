---
title: "ELF 文件结构分析"
date: 2023-05-28T15:59:11+08:00
lastmod: 2023-05-28T15:59:11+08:00 #更新时间
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
## 目标文件的格式
目前，Linux 平台流行的 **可执行文件**（Executable）主要包含以下格式：
- Linux 下的 **ELF（Executable Linkable Format）**，注意这里是二进制文件其内容的组织格式，与后缀无关；

目标文件是源代码经过编译后但是未进行链接的那些中间文件（Linux 下为 `.o` 文件），它与可执行文件格式非常相似，一般与可执行文件一起采用同一种格式存储，Linux 下采用 ELF 文件格式。

动态链接库（Dynamic Linking Library）、静态链接库（Static Linking Library）均采用可执行文件格式存储，Linux 下均按照 ELF 格式存储。
- Linux 下的 `.so`、`.a`；

## ELF 文件结构
CSAPP 上的 ELF 格式文件结构图：

![8A6Nume3zbD4rfk](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065839.png)

更详细的 ELF 文件结构图：

![r6qALmx8dRVwPhD](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065842.png)

可以看到，ELF 文件包含四个部分：
- 第一部分为 ELF Header；
- 第二部分为 Program Header Table，**Relocatable object file** 中该部分不存在，**Executable object file** 中该部分存在；
- 第三部分为 ELF Sections，包括 `.text`、`.rodata`、`.data`、`.bss`等；
- 第四部分为 ELF Section Header Table（或称节头表，后面以 sht 指代），注意 sht 不像 ELF Header 那样只有一块，它由多个 Section header table entry 组成。

## ELF 的 16 进制内容
`elf.c` 的内容如下：

```c
unsigned long long data1 = 0xdddddddd11111111;
unsigned long long data2 = 0xdddddddd22222222;
void func1() {
}
void func2() {
}
```

执行 `gcc -c elf.c`，会生成 `elf.o`，执行 `hexdump elf.o > elf.txt`，会将 `elf.o` 的内容以 16 进制形式存储在 `elf.txt` 中，`elf.txt` 的内容如下：

```txt
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  01 00 3e 00 01 00 00 00  00 00 00 00 00 00 00 00  |..>.............|
00000020  00 00 00 00 00 00 00 00  28 02 00 00 00 00 00 00  |........(.......|
00000030  00 00 00 00 40 00 00 00  00 00 40 00 0b 00 0a 00  |....@.....@.....|
00000040  55 48 89 e5 90 5d c3 55  48 89 e5 90 5d c3 00 00  |UH...].UH...]...|
00000050  11 11 11 11 dd dd dd dd  22 22 22 22 dd dd dd dd  |........""""....|
00000060  00 47 43 43 3a 20 28 44  65 62 69 61 6e 20 31 32  |.GCC: (Debian 12|
00000070  2e 32 2e 30 2d 31 34 29  20 31 32 2e 32 2e 30 00  |.2.0-14) 12.2.0.|
00000080  14 00 00 00 00 00 00 00  01 7a 52 00 01 78 10 01  |.........zR..x..|
00000090  1b 0c 07 08 90 01 00 00  1c 00 00 00 1c 00 00 00  |................|
000000a0  00 00 00 00 07 00 00 00  00 41 0e 10 86 02 43 0d  |.........A....C.|
000000b0  06 42 0c 07 08 00 00 00  1c 00 00 00 3c 00 00 00  |.B..........<...|
000000c0  00 00 00 00 07 00 00 00  00 41 0e 10 86 02 43 0d  |.........A....C.|
000000d0  06 42 0c 07 08 00 00 00  00 00 00 00 00 00 00 00  |.B..............|
000000e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000000f0  01 00 00 00 04 00 f1 ff  00 00 00 00 00 00 00 00  |................|
00000100  00 00 00 00 00 00 00 00  00 00 00 00 03 00 01 00  |................|
00000110  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000120  07 00 00 00 11 00 02 00  00 00 00 00 00 00 00 00  |................|
00000130  08 00 00 00 00 00 00 00  0d 00 00 00 11 00 02 00  |................|
00000140  08 00 00 00 00 00 00 00  08 00 00 00 00 00 00 00  |................|
00000150  13 00 00 00 12 00 01 00  00 00 00 00 00 00 00 00  |................|
00000160  07 00 00 00 00 00 00 00  19 00 00 00 12 00 01 00  |................|
00000170  07 00 00 00 00 00 00 00  07 00 00 00 00 00 00 00  |................|
00000180  00 65 6c 66 2e 63 00 64  61 74 61 31 00 64 61 74  |.elf.c.data1.dat|
00000190  61 32 00 66 75 6e 63 31  00 66 75 6e 63 32 00 00  |a2.func1.func2..|
000001a0  20 00 00 00 00 00 00 00  02 00 00 00 02 00 00 00  | ...............|
000001b0  00 00 00 00 00 00 00 00  40 00 00 00 00 00 00 00  |........@.......|
000001c0  02 00 00 00 02 00 00 00  07 00 00 00 00 00 00 00  |................|
000001d0  00 2e 73 79 6d 74 61 62  00 2e 73 74 72 74 61 62  |..symtab..strtab|
000001e0  00 2e 73 68 73 74 72 74  61 62 00 2e 74 65 78 74  |..shstrtab..text|
000001f0  00 2e 64 61 74 61 00 2e  62 73 73 00 2e 63 6f 6d  |..data..bss..com|
00000200  6d 65 6e 74 00 2e 6e 6f  74 65 2e 47 4e 55 2d 73  |ment..note.GNU-s|
00000210  74 61 63 6b 00 2e 72 65  6c 61 2e 65 68 5f 66 72  |tack..rela.eh_fr|
00000220  61 6d 65 00 00 00 00 00  00 00 00 00 00 00 00 00  |ame.............|
00000230  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000260  00 00 00 00 00 00 00 00  1b 00 00 00 01 00 00 00  |................|
00000270  06 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000280  40 00 00 00 00 00 00 00  0e 00 00 00 00 00 00 00  |@...............|
00000290  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
000002a0  00 00 00 00 00 00 00 00  21 00 00 00 01 00 00 00  |........!.......|
000002b0  03 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000002c0  50 00 00 00 00 00 00 00  10 00 00 00 00 00 00 00  |P...............|
000002d0  00 00 00 00 00 00 00 00  08 00 00 00 00 00 00 00  |................|
000002e0  00 00 00 00 00 00 00 00  27 00 00 00 08 00 00 00  |........'.......|
000002f0  03 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000300  60 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |`...............|
00000310  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
00000320  00 00 00 00 00 00 00 00  2c 00 00 00 01 00 00 00  |........,.......|
00000330  30 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |0...............|
00000340  60 00 00 00 00 00 00 00  20 00 00 00 00 00 00 00  |`....... .......|
00000350  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
00000360  01 00 00 00 00 00 00 00  35 00 00 00 01 00 00 00  |........5.......|
00000370  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000380  80 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000390  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
000003a0  00 00 00 00 00 00 00 00  4a 00 00 00 01 00 00 00  |........J.......|
000003b0  02 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000003c0  80 00 00 00 00 00 00 00  58 00 00 00 00 00 00 00  |........X.......|
000003d0  00 00 00 00 00 00 00 00  08 00 00 00 00 00 00 00  |................|
000003e0  00 00 00 00 00 00 00 00  45 00 00 00 04 00 00 00  |........E.......|
000003f0  40 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |@...............|
00000400  a0 01 00 00 00 00 00 00  30 00 00 00 00 00 00 00  |........0.......|
00000410  08 00 00 00 06 00 00 00  08 00 00 00 00 00 00 00  |................|
00000420  18 00 00 00 00 00 00 00  01 00 00 00 02 00 00 00  |................|
00000430  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000440  d8 00 00 00 00 00 00 00  a8 00 00 00 00 00 00 00  |................|
00000450  09 00 00 00 03 00 00 00  08 00 00 00 00 00 00 00  |................|
00000460  18 00 00 00 00 00 00 00  09 00 00 00 03 00 00 00  |................|
00000470  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000480  80 01 00 00 00 00 00 00  1f 00 00 00 00 00 00 00  |................|
00000490  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
000004a0  00 00 00 00 00 00 00 00  11 00 00 00 03 00 00 00  |................|
000004b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000004c0  d0 01 00 00 00 00 00 00  54 00 00 00 00 00 00 00  |........T.......|
000004d0  00 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
000004e0  00 00 00 00 00 00 00 00                           |........|
000004e8
```

在 `/usr/include/elf.h` 中，有两个结构体 `Elf64_Ehdr` 和 `Elf64_Shdr`，分别对应 ELF Header 和 Section header table entry，我们根据这两个结构体的声明，就能大致解析 `elf.txt` 文件。

```c
typedef uint16_t Elf64_Half;
typedef uint32_t Elf64_Word;
typedef	int32_t  Elf64_Sword;
typedef uint64_t Elf64_Xword;
typedef	int64_t  Elf64_Sxword;
typedef uint64_t Elf64_Addr;
typedef uint64_t Elf64_Off;
typedef struct
{
  Elf64_Word	sh_name;		/* Section name (string tbl index) */
  Elf64_Word	sh_type;		/* Section type */
  Elf64_Xword	sh_flags;		/* Section flags */
  Elf64_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf64_Off	sh_offset;		/* Section file offset */ 为 0x0e = 14
  Elf64_Xword	sh_size;		/* Section size in bytes */
  Elf64_Word	sh_link;		/* Link to another section */
  Elf64_Word	sh_info;		/* Additional section information */
  Elf64_Xword	sh_addralign;		/* Section alignment */
  Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr

typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Elf64_Half	e_type;			/* Object file type */
  Elf64_Half	e_machine;		/* Architecture */
  Elf64_Word	e_version;		/* Object file version */
  Elf64_Addr	e_entry;		/* Entry point virtual address */
  Elf64_Off	e_phoff;		/* Program header table file offset */
  Elf64_Off	e_shoff;		/* Section header table file offset */
  Elf64_Word	e_flags;		/* Processor-specific flags */
  Elf64_Half	e_ehsize;		/* ELF header size in bytes */
  Elf64_Half	e_phentsize;		/* Program header table entry size */
  Elf64_Half	e_phnum;		/* Program header table entry count */
  Elf64_Half	e_shentsize;		/* Section header table entry size */
  Elf64_Half	e_shnum;		/* Section header table entry count */
  Elf64_Half	e_shstrndx;		/* Section header string table index */
} Elf64_Ehdr;
```

## 解析 ELF 文件
`elf.txt` 第一行 `00000000` 对应的内容为 **magic number**，又称**魔数**，标识该文件格式，例如 `0x7f` 说明是 `ELF` 格式，`0x45`、`0x4c`、`0x46` 分别是 `E`、 `L`、`F` 的 ascii 码。

第二行 `00000010` 前 8 个字节分别表示 `e_type`、`e_machine`、`e_version`，`e_type` 为 `0x01` 表示为 **Relocatable object file**，`e_machine` 为 `0x3e` 表示为 `x86_64` 机器，后 8 个字节为 `e_entry`，规定 `ELF` 程序的入口虚拟地址，操作系统在加载完程序之后从这个地址开始执行进程的指令，**Relocatable object file** 一般没有入口地址，该值为 0。

第三行 `00000020` 前 8 个字节为 `e_phoff`，表示文件中 **Program Header Table** 的偏移量，对 **Relocatable object file** 来说无意义，为 0；后 8 个字节为 `e_shoff`，注意是小端机，其值为 `0x228`，说明 sht 的偏移量为 `0x228`（也可以认为从 `0x228` 处起为 sht）。

第四行 `00000040` 前 4 个字节为 `e_flags`，我们不关心；5、6 字节为 `e_ehsize`，为 `0x40`，说明 ELF Header 占据大小为 40 字节；7、8 字节为 `e_phentsize`，为每个 Program header table entry 占据的大小，由于是 **Relocatable object file**，这里为 0；9、10  字节为 Program header table entry 的个数，也为0；11、12 字节为 **sht entry** 的大小，为 `0x40`，即 64 字节；13、14 字节为 **sht entry** 的个数，为 `0x0b`；最后两个字节为 `e_shstrndx`，值为 `0xa`，指示了节名称字符串表在 **sht** 中的位置。

到这里，前四行的内容即为 **ELF Header** 的所有内容，我们已经可以计算出该 ELF 格式文件的大小了，为
$size = e\_shoff + e\_shentsize * e\_shnum = \texttt{0x228} + \texttt{0x40} * \texttt{0x0b} = \texttt{0x4e8}$，`elf.txt` 文件的最后一行正好是 `0x4e8`。

我们也可以执行 `readelf -h elf.o`，其内容如下：

```txt
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          552 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         11
  Section header string table index: 10
```

与我们之前的分析也能够对应起来。

## Section header table 解析

首先，执行 `readelf -S elf.o`，输出如下：
```sh
There are 11 section headers, starting at offset 0x228:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000000e  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  00000050
       0000000000000010  0000000000000000  WA       0     0     8
  [ 3] .bss              NOBITS           0000000000000000  00000060
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .comment          PROGBITS         0000000000000000  00000060
       0000000000000020  0000000000000001  MS       0     0     1
  [ 5] .note.GNU-stack   PROGBITS         0000000000000000  00000080
       0000000000000000  0000000000000000           0     0     1
  [ 6] .eh_frame         PROGBITS         0000000000000000  00000080
       0000000000000058  0000000000000000   A       0     0     8
  [ 7] .rela.eh_frame    RELA             0000000000000000  000001a0
       0000000000000030  0000000000000018   I       8     6     8
  [ 8] .symtab           SYMTAB           0000000000000000  000000d8
       00000000000000a8  0000000000000018           9     3     8
  [ 9] .strtab           STRTAB           0000000000000000  00000180
       000000000000001f  0000000000000000           0     0     1
  [10] .shstrtab         STRTAB           0000000000000000  000001d0
       0000000000000054  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

首先，从之前的 ELF Header 我们可以得知，sht 是从 `0x228` 开始的，sht entry 的 size 为 `0x40`，第零号 `entry` 的 `name` 为空，其他值也全都是 0，我们可以看第一号 `entry`。

前 4 个字节为 `sh_name`，值为 `0x1b`，表示的是它所代表的字符串在 `section header string table` 中的起始位置的偏移（或者说索引），我们可以看到，`.shstrtab` 的 `offset` 为 `0x1d0`，`0x1d0 + 0x1b` 的位置，以该位置起始的字符串为 `0x2e`、`0x74`、`0x65`、`0x78`、`0x74`，其 ascii 码对应的字符组成的字符串为 `.text`。

5~8 四个字节为 `sh_type`，值为 `0x1`；

9～16 这八个字节为 `sh_flags`，值为`0x6`，对应二进制为 $110_{(2)}$，要从二进制掩码的角度考虑这个值，通过 `sh_flags` 和 `sh_type` 即可确定该 Section 是 `PROGBITS` 类型，具有可写和可分配的属性；

17～24 这八个字节为 `sh_addr`，为 Section 在内存中的虚拟地址，对于 **Relocatable object file** 来说，该值没什么意义，因此为 0；

25～32 这八个字节为 `sh_offset`，即该 sht 对应的 **Section table** 的在文件中的 `offset`，值为 `0x40`；

33～40 这八个字节为 `sh_size`，表示 **Sectionn table** 的大小，这里为 `0x0e`；

41～48 这八个字节为 `sh_link` 和 `sh_info`，这里对其意义暂时不予讨论；

49～56 这八个字节为 `sh_addralign`，表示节头表的对齐方式；

59～64 这八个字节为 `sh_entsize`，表示 **Section table** 的单个表项的大小，这里值为 `0x0`，没有意义；

我们可以查看 `.symtab` sht 的 `sh_entsize` 为 `0x18`，其 `sh_size` 为 `0xa8`，$\frac{\texttt{0xa8}}{\texttt{0x18}} = \texttt{0x7}$，所以 `.symtab` Section table 一共有七个表项。

执行 `readelf -s elf.o`，输出为

```sh
Symbol table '.symtab' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS elf.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 .text
     3: 0000000000000000     8 OBJECT  GLOBAL DEFAULT    2 data1
     4: 0000000000000008     8 OBJECT  GLOBAL DEFAULT    2 data2
     5: 0000000000000000     7 FUNC    GLOBAL DEFAULT    1 func1
     6: 0000000000000007     7 FUNC    GLOBAL DEFAULT    1 func2
```

输出与计算结果一致。

## 参考
[计算机那些事(4)——ELF文件结构](http://chuquan.me/2018/05/21/elf-introduce/)

[CSAPP](https://csapp.cs.cmu.edu/)

[yangaaamin](https://www.bilibili.com/video/BV17K4y1N7Q2/)