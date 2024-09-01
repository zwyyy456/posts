---
title: "计算机教育缺失的一课：Shell 工具与脚本"
date: 2024-05-09
lastmod: 2024-05-11T11:38:39+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cs"]
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
## `$` 符号的功能

Shell 中，`$` 符号可以与数字或者其他符号组合在一起表示特殊的值或者变量，如下：

```sh
#!/bin/bash
echo $0 # $0 用于获取当前脚本文件名名称
echo "The first parameter: $1" # $1 表示第一个参数
echo "The eleventh parameter: ${11}2"
echo "The eleventh parameter: ${11}x"
echo "The eleventh parameter: $11x"
echo "The eleventh parameter: $12"
tmp="hello"
echo "$tmp world"
```

将以上脚本文件命名为 `test.sh` 并执行 `bash test.sh 1 2 3 4 5 6 7 8 9 10 13 14`，输出如下：

```txt
test.sh
The first parameter: 1
The eleventh parameter: 132
The eleventh parameter: 13x
The eleventh parameter: 11x
The eleventh parameter: 12
hello wolrd
```

这是因为，只有 `$n` 才能表示第 `n` 个参数（`n` 为单个数字），如果想表示第 `11` 个参数，就必须使用 `${11}`（加上大括号）来表示，bash 会将 `$12` 处理为名为 `12` 的变量。

此外，`$#` 表示参数的个数，类似于程序中的 `argc`，`$@` 表示所有参数的列表，在 bash 或者 zsh 中，`$_` 表示上一个命令的最后一个参数，而 `!!` 则表示执行的上一条命令，`$?` 表示上一个命令的返回状态，为 0 表示命令成功执行。

例如：

```sh
mkdir tmp
cd $_ # 进入到了 tmp 目录
mkdir test
sudo !! # 这里就是表示执行 sudo mkdir test
```

## shell 中定义函数与使用

执行 `nvim mcd.sh`，将其内容修改为如下：

```
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

然后保存，再执行 `source mcd.sh`，即在当前 shell 执行了 `mcd.sh`，而 `./mcd.sh` 会创建一个子 shell 来执行，执行了 `source mcd.sh` 之后，当前 shell 中就有一个我们定义的名为 `mcd` 的函数了，在当前 shell 中，我们就可以使用 `mcd` 命令来创建并进入指定的目录了：

> 注意，这里不能用 `fish`，必须用 `bash` 或者兼容 `bash` 的 shell 例如 `fish`。

```sh
source mcd.sh
mcd arm # 将进入创建的名为 arm 的目录中
```

## 逻辑表达式

shell 也支持逻辑表达式的与、或，如下图所示：

![H7c3Xp](https://pic-upyun.zwyyy456.tech/uPic/H7c3Xp.png)

对于“或”运算，如果左侧的结果已经为真，那么就不会再去计算右侧的结果，因此 `echo "Oops fail"` 无输出，“与”运算的原理相同。

## 变量

当使用双引号来表示字符串时，shell 会对引号内的内容进行变量与命令替换，它将 `()` 包裹的内容视为命令，将 `{}` 包裹的内容视为变量，并进行替换，将 `()` 包裹的内容替换为输出结果，将 `{}` 包裹的内容替换为变量的值。

```shell
zwyyy in 🌐 armbian in ~/missing-semester/arm
❯ foo=$(pwd)

zwyyy in 🌐 armbian in ~/missing-semester/arm
❯ echo $foo
/home/zwyyy/missing-semester/arm

zwyyy in 🌐 armbian in ~/missing-semester/arm
❯ echo "We are in $(pwd)"
We are in /home/zwyyy/missing-semester/arm

zwyyy in 🌐 armbian in ~/missing-semester/arm
❯ echo 'We are in $(pwd)'
We are in $(pwd)

zwyyy in 🌐 armbian in ~/missing-semester/arm
❯ echo "we are in ${foo}"
we are in /home/zwyyy/missing-semester/arm
```


以下是一个使用到了前面所述的知识点的脚本：

```sh
#! /bin/bash

echo "Starting program at $(date)"

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    if [[ "$?" -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foorbar" >> "$file"
    fi
done
```

其中，`grep foobar "$file" > /dev/null 2> /dev/null


## shell 的一些缩写

`convert image.{png,jpg}` 等价于 `convert image.png image.jpg`；

而 `touch foo{,1,2,10}` 等价于 `touch foo foo1 foo2 foo10`；

而 `touch project{1,2}/src/test/test{1,2,3}.py` 等价于 `touch project1/src/test/test1.py project1/src/test/test2.py project1/src/test/test3.py project2/src/test/test1.py project2/src/test/test2.py project2/src/test/test3.py`。


## 与 Python 交互的 shell 脚本

```py
#!/usr/bin/python3

# test.py
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)

# foobar
```

其中，第一行注释 `#!/usr/bin/python3` 被称为 `shebang`，用于指定 python 脚本该由哪个解释器运行。

> 我们当然可以通过 `python test.py` 来运行该脚本，此时不需要 `shebang`，但假设我们希望通过 `./test.py` 来运行，那么久需要 `shebang` 来告诉 shell，应该使用哪个 python 解释器来运行该脚本。

将 `#!/usr/bin/python3` 替换为 `#!/usr/bin/env python3` 能增强程序的可移植性，`/usr/bin/env` 是一个 Unix 程序，它会查找 `python3` 解释器的路径并执行它，因此，对于我的电脑来说，`#!/usr/bin/env python3` 就相当于 `#!/usr/bin/env python3`。


