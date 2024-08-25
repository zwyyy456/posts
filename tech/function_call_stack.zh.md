---
title: "从汇编的角度理解 C/Cpp 的函数调用过程"
date: 2023-05-01T17:45:58+08:00
lastmod: 2023-05-01T17:45:58+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["OS"]
tags: ["Cpp"]
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
## 代码
测试代码内容如下，定义了一个 `add` 函数，用来求两个函数的和。
```c
int add(int a, int b) {
    return a + b;
}
int sum(int a, int b) {
    return 10 + add(a, b);
}
int main() {
    int res = sum(10, 20);
    return 0;
}
```

汇编代码如下:
```asm
	.file	"add.c"
	.text
	.globl	add
	.type	add, @function
add:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	%edi, -4(%rbp)
	movl	%esi, -8(%rbp)
	movl	-4(%rbp), %edx
	movl	-8(%rbp), %eax
	addl	%edx, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	add, .-add
	.globl	sum
	.type	sum, @function
sum:
.LFB1:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$8, %rsp
	movl	%edi, -4(%rbp)
	movl	%esi, -8(%rbp)
	movl	-8(%rbp), %edx
	movl	-4(%rbp), %eax
	movl	%edx, %esi
	movl	%eax, %edi
	call	add
	addl	$10, %eax
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE1:
	.size	sum, .-sum
	.globl	main
	.type	main, @function
main:
.LFB2:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$16, %rsp
	movl	$2, %esi
	movl	$1, %edi
	call	sum
	movl	%eax, -4(%rbp)
	movl	$0, %eax
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE2:
	.size	main, .-main
	.ident	"GCC: (Debian 12.2.0-14) 12.2.0"
	.section	.note.GNU-stack,"",@progbits
```

我们先分析 `main()` 函数的汇编代码，关注主要部分：
```asm
pushq %rbp ; 将原先的 rbp 再入栈
movq %rsp, %rbp ; 即rbp = rsp，相当于让 rbp 也指向栈顶
subq	$16, %rsp ; 从内存中分配空间给栈，因为栈是从高地址向低地址扩展，因此是 subq
movl	$2, %esi
movl	$1, %edi
call	add
```
`rbp` 表示的是 64 位系统环境下的栈基地址寄存器，`rsp` 表示栈顶地址寄存器。完成分配内存后，将函数实参存入对应的寄存器。

## 栈帧
我们可以把函数调用抽象成栈中的一个元素，这个元素就被称为栈帧（stack frame）。开始时，`rbp` 中其实是上一个栈帧的的地址，可以看到 `main()` 和 `add()` 都是先执行了 `pushq %rbp`，一个新的栈帧就生成了，此时 `rsp` 就指向这个栈帧（可以认为这个栈帧里面此时只有一个元素），然后执行 `movq %rsp, %rbp`，即让 `rbp` 中的数据变成这个新的栈帧的栈底，然后开始执行 `subq $16, %rsp`，由于栈在内存中是由高地址向低地址扩张，因此是 `subq`。
![5A1bOhlJFaMWydS](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065913.jpg)
调用者栈帧的地址会比被调用者栈帧的地址小。
图片的上半部分是调用者的栈帧，可以看到里面存有参数（也就是一种局部变量）。也有当前函数的返回地址（不是 `return` 语句），通过这个地址可以找到当前这个函数运行完了应该返回哪里。
返回地址是通过当前栈帧的帧指针（即 `rbp` 中的内容）确定的，它总是储存在当前帧指针 +8 的位置（在 64 位机器中，如果是图中的 32 位，那就是 +4 的位置）。
下半部分存的是当前函数的栈帧，里面同样存有局部变量。ebp 和 esp 分别标注了这个栈帧的起始和结束位置。通过帧指针加上一些偏移量，就可以访问到这个栈帧里的局部变量。

## 调用函数时栈帧的变化
1. 调用一个函数时，我们先把函数的返回地址（也就是执行调用时 pc 的值）压入栈中（是否可以认为栈帧的栈顶存储函数的返回地址？）；
    > pc 即 program counter, 程序计数器，它指向下一条指令所在的内存单元的地址，通过 pc，计算机总是可以知道下一步该干什么。
    > `call add` 相当于一次做了两件事情，把 `call` 指令执行时的 pc 压入栈中，然后把 pc 的值改成 `add` 函数的起始地址。
2. 将旧的 `rbp` 的值压入栈中，此时可以认为已经到了一个新的栈帧中了，`rsp` 指向这个新的栈帧，但是 `rbp` 还是指向上一个栈帧的栈底；
3. `movq %rsp, %rbp`：让 `rbp` 指向这个新的栈帧；
4. `subq $16, %rsp`：更新栈顶指针的值，扩充这个新的栈帧；
5. 栈帧已经有了足够的空间，可以放入局部变量并执行这个函数了，至此，新帧的插入全部完成。

## 函数返回时栈帧的变化
1. 释放之前占用的内存，因此直接把 `rsp` 的值设为 `rbp` 的值，相当于移动了栈顶指针；
    > `leave` 会做两件事，将栈帧的栈顶指针指向栈底；恢复备份的栈帧的栈底指针；（相当于`movq %rbp, %rsp` 和 `pop %rbp` 的结合）
2. 从栈中弹出栈帧的栈底指针；
3. 弹出返回地址，赋值到 pc；
4. 根据 pc 的值，继续执行原函数；

## 参考内容
[函数调用是如何实现的](https://ttzytt.com/2022/04/function-call/#fn:2)

