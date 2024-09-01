---
title: "利用 Latex 写毕业论文"
date: 2024-02-22T20:18:43+08:00
lastmod: 2024-02-22T20:18:43+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["latex"]
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
## 前言

本科写毕业论文的时候，是用的学校的 word 模板，体验比较糟糕，当时跟我一组的同学就是用的 Latex 的本科学位论文模板，当时由于我写本科毕业论文的时间比较紧迫了，所以没有折腾这些，此外，当时的 Latex 学位论文模板属于是纯民间支持的性质，而现在，学校已经半官方地支持了 Latex 学位论文模板，因此，研究生毕业论文我就准备使用 Latex 而不再使用 word 来写了。

## 模板介绍

首先观察 `main.tex`，其中 `\input{contents/abstract}` 就是表示会加载 `main.tex` 所在目录的的 `contents` 目录下的 `abstract.tex` 的内容，即摘要。

然后是 `\tablecontents` 命令以生成目录，目录生成方式应该是由模板决定的。

以下这些则是正文内容：

```tex
\input{contents/intro}
\input{contents/math_and_citations}
\input{contents/floats}
\input{contents/summary}
```

我们可以观察一下 `contents` 目录下的 `intro.tex`，首先是 `\chapter{简介}`，这里的大括号中的内容就是这一章的标题，后续则分别是一级标题、二级标题、三级标题、四级标题，对应的 Latex 语法如下：

```tex
\section{二级标题}
\subsection{三级标题}
\subsubsection{四级标题}
```

模板的默认字体应该就是宋体，模板中给出了手动指定该段落的字体的方法。

`contents` 目录下的 `math_and_citations.tex` 给出了 Latex 中的数学公式（包括字符）与引用文献的标注方法。

`contents` 目录下的 `floats.tex` 则是给出了 Latex 插入图片与表格的方法。

`setup.tex` 中则是封面以及 Latex 的样式控制的相关信息。

## SJTUThesis 模板食用方法

### Overleaf

学校网络信息中心应该是根据 Overleaf 自己搭建了一个可以在线编译 Latex 的平台：[Latex 文档助手](https://latex.sjtu.edu.cn/)。

首先去 [SJTUThesis](https://github.com/sjtug/SJTUThesis) 的 GitHub 页面下载论文模板的 [压缩包：master.zip](https://github.com/sjtug/SJTUThesis/archive/refs/heads/master.zip)，在平台上点击 `创建新项目 -> 上传项目`，如下图所示：

![BalA0r](https://pic-upyun.zwyyy456.tech/uPic/BalA0r.png)

点击项目，可以自行重命名，记得要在菜单处设置编译器为 `XeLatex`，默认的 `pdfLaTex` 会导致无法编译生成 PDF，如下图所示：

![uZbF2X](https://pic-upyun.zwyyy456.tech/uPic/uZbF2X.png)

> 对于 Windows，如果安装了 IDM，那么需要取消 IDM 对该网站的下载的自动接管，否则会导致每次编译的时候，网站编译失败，无法在网站查看 PDF，同时 IDM 会提示你是否下载 PDF（该 PDF 是正常的编译结果）。

### VsCode

学校搭建的在线平台虽然能在线同步文档内容，省去了自行配置 Latex 环境，但仍有一些不方便的地方，例如要插入图片，需要自己手动上传 `xxx.pdf`，参考文献也是，并且编译速度较慢，今天使用的时候还碰到了服务器问题，故选择在自己的 Winodws 笔记本上搭建本地的 Latex 编译环境。

首先，执行 `scoop install latex`，则 scoop 会为你自动安装 MiKTeX，再执行 `scoop install perl` 以安装必须的工具。MiKTex 与 Tex Live 是 Latex 的两种发行版，MiKTeX 支持 `install on the fly`，即用到某个 package 时再进行安装，因此初次安装时，体积会比大而全的 TeX Live 小很多。当然，最关键的原因是，scoop 只能安装 MiKTeX。

安装好之后，在 VsCode 中还需要安装 LaTeX Workshop 插件，安装好之后，对于交大的毕业论文模板，只需要在编译的时候选择 `Recipe: latexmk(xelatex)` 编译即可，并在设置中将 `latex-workshop.latex.recipe.default` 修改为 `lastUsed` 以一直使用该选项来编译。

> 不知道换用其他模板之后是否还是这样，不过除了写论文，以后用到 LaTeX 的机会应该也挺少的。

之后，打开 MiKTeX 的设置，将 `自动安装缺失的宏包` 的选项设置为总是，如下图：

![](https://pic-upyun.zwyyy456.tech/picgo/20240427155059.png)

然后在 VsCode 中打开该模板项目，终端中执行 `./Compile.bat`，就会编译生成 PDF，这个过程中 MiKTeX 会自动安装需要的宏包。

VsCode 中，快捷键 `Ctrl + Alt + b` 是编译 LaTeX 项目生成 PDF 的快捷键，PDF 生成目录可能需要执行两次编译。

## 将 LaTeX 部署到服务器上

### 安装 MiKTex

由于博主平时使用的主力电脑是 MacBook 14 Pro，而之前的 Windows 笔记本存在每隔几天就可能死机的问题，因此博主本来是希望将 LaTeX 也部署在自己的 mac 上的，后面想到自己宿舍有一台 J3855u 的小主机（还有几个吃灰的 VPS），于是博主向 ChatGPT 咨询了一下 LaTeX 的性能要求，得出的结论是 J3855u 足以满足，因此干脆装在刷了 Debian 的小主机上，mac 通过 VsCode Remote 连接上去来编写。

由于 Tex Live 体积过大，有很多不必要的宏包，因此这里还是选择安装 MiKTex。安装教程可以参见 [MiKTex 官方安装教程](https://miktex.org/download)，执行以下命令即可安装：

```sh
curl -fsSL https://miktex.org/download/key | sudo tee /usr/share/keyrings/miktex-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/miktex-keyring.asc] https://miktex.org/download/debian bookworm universe" | sudo tee /etc/apt/sources.list.d/miktex.list
sudo apt-get update
sudo apt-get install miktex
```

执行完命令之后，安装还剩最后一步，考虑到我的 Debian 小主机没有 GUI，因此这里使用命令行来操作：

```sh
miktexsetup finish # 完成安装，这里是安装给了当前用户，如果希望为所有用户安装，请执行 `sudo miktexsetup --shared=yes finish`
initexmf --set-config-value [MPM]AutoInstall=1 # 设置自动安装缺失的宏包，如果前面选择了为所有用户安装，则应该执行 `sudo initexmf --admin --set-config-value [MPM]AutoInstall=1`
```

这里要注意的是，如果只为当前用户安装，那么需要将 `~/bin` 添加到环境变量中，由于每个人使用的 Shell 不同，添加环境变量的方法也可能不同，这里不再赘述。

安装好之后，再安装 VSCode 的 LaTeX Workshop 插件，使用方法与前文一致。

### 使用自定义字体

完成安装之后，执行 `git clone https://github.com/sjtug/SJTUThesis.git` 即可获取模板，`cd SJTUThesis` 之后，执行 `make all` 理论上即可编译生成 PDF，然而，在 Debian 上，该模板默认使用的中文字体是 `Fandol`，该字体年久失修，缺少许多中文字符，参照 [在 Overleaf 上配置自定义中文字体](https://github.com/sjtug/SJTUThesis/wiki/%E5%9C%A8%E7%BA%BF%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E)。

首先找到 Windows 系统自带的「宋体」、「黑体」、「楷体」、「仿宋」的字体文件，一般存放在 `C:\Windows\Fonts` 目录下，文件名分别为 `simsun.ttc`、`simhei.ttf`、`simkai.ttf`、`simfang.ttf`。

我将它们放在了 `~/code/latex/fonts` 目录下，模板项目位于 `~/code/latex/master-thesis` 目录，然后在 `main.tex` 中配置字体，首先在模版类选项中添加 `cjk-font=none` 来关闭 `ctex` 的自动中文配置：

```tex
\documentclass[type=master, cjk-font=none]{sjtuthesis}
```

然后在 `input{setup}` 之前添加以下内容：

```tex
% 设置字体路径
\defaultfontfeatures{Path=../fonts/}

% 设置中文字体
\setCJKmainfont[
  AutoFakeBold = 3,
  ItalicFont   = simkai.ttf
]{simsun.ttc}
\setCJKsansfont[AutoFakeBold=3]{simhei.ttf}
\setCJKmonofont{simfang.ttf}
\setCJKfamilyfont{zhsong}{simsun.ttc}[
  AutoFakeBold = 3,
  ItalicFont   = simkai.ttf
]
\setCJKfamilyfont{zhhei}{simhei.ttf}[AutoFakeBold=3]
\setCJKfamilyfont{zhkai}{simkai.ttf}
\setCJKfamilyfont{zhfs}{simfang.ttf}

\newcommand*{\songti}{\CJKfamily{zhsong}}
\newcommand*{\heiti}{\CJKfamily{zhhei}}
\newcommand*{\kaishu}{\CJKfamily{zhkai}}
\newcommand*{\fangsong}{\CJKfamily{zhfs}}
```

之后再执行 `make all` 来编译生成 PDF 时，就不会报没有对应中文字符的错误了。

另外还有一种方法，就是先执行以下命令，将 Windows 的这些中文字体安装到系统：

```sh
sudo mkdir /usr/share/fonts/winfonts
sudo cp *.ttf /usr/share/fonts/winfonts # *.ttf 即你要安装的字体
sudo cp *.ttc /usr/share/fonts/winfonts
sudo mkfontscale
sudo mkfontdir
sudo fc-cache -fsv
```

再查看系统中安装的中文字体：

```sh
fc-list :lang=zh | grep sim
```

有输出，说明中文字体已经安装好了，然后修改模板类选项为 `\documentclass[type=master, cjk-font=windows]{sjtuthesis}` 即可。

![8ab2f72e47c9abf1d814f07f27be6341](https://pic-upyun.zwyyy456.tech/uPic/8ab2f72e47c9abf1d814f07f27be6341.jpg)



## 插入参考文献

我使用的文献管理软件是 zotero，要插入参考文献到 Latex 可以说非常方便，在 zotero 中，选中你要插入的那些文献，`右键` -> `导出条目`，如下图所示：

![](https://pic-upyun.zwyyy456.tech/picgo/20240421113130.png)

然后选择格式为 BibLaTex，如下图所示，

![](https://pic-upyun.zwyyy456.tech/picgo/20240421115332.png)

之后则会导出生成一个 `xxx.bib` 文件，然后，我在我的 Latex 项目中新建了一个 `myref` 目录，将 `xxx.bib` 文件导入到了该目录中，并在 `setup.tex` 中添加 `addbibiresourse{myref/xxx.bib}`，然后就可以在正文中通过 `\cite{x1}` 来进行引用了，其中 `x1` 是你要引用的文章在 `xxx.bib` 中的 id，即 `@article{` 后面跟的内容（或者 `@thesis{` 等），如下图所示：

![](https://pic-upyun.zwyyy456.tech/picgo/20240421115918.png)

> 注意，如果 `setup.tex` 中添加了 `addbibiresourse{myref/xxx.bib}` 内容，而对应目录不存在 `xxx.bib` 文件，则无法生成 `main.bbl`，从而生成的 PDF 的中所有参考文献都无法正常插入！

至于其他格式的参考文献的插入，可以参见交大的 Latex 模板中的说明。

Better BibTex 首选项中，`导出` -> `字段`设置为 `abstract, file, url`，即导出时不包括这三个字段。



## 插入图片

~~Latex 中插入图片稍微有点麻烦，由于我的 Latex 的编译器是 XeLaTex，因此要插入的图片必须保存为 `.eps` 格式，对于 Windows 来说，可以执行 `scoop install imagemagick` 来下载安装图片转换软件，安装好之后，终端中进入图片所在的目录，执行 `magick convert xxx.png xxx.eps` 即可将 `.png` 格式的图片转换为 `.eps` 格式。~~

~~然后，在 Latex 项目中新建一个 `mypic` 目录，将 `xxx.eps` 导入到该目录中，在你需要插入图片的正文处，添加以下代码：~~

事实上，eps 已经是较为落后的格式了，容易出现奇怪的兼容性问题，导致无法成功编译生成 PDF。我在 Debian 上就碰到了样的问题，目前推荐将图片转换成 PDF 再插入到 LaTeX 中，而不是转换成 eps。Debian 上执行 `sudo apt-get install imagemagick` 安装好 imagemagick 之后，执行 `convert xxx.png xxx.pdf` 就能将 png 格式的图片转换成 PDF 格式。

```tex
\begin{figure}[!htp]
    \centering
    \includegraphics[]{mypic/detect_along_edge.eps} % 大括号中为图片路径，中括号中可以添加图片的 height、weight 等大小参数
    \bicaption{沿边缘探测}{detection along the edge} % 图片的标题
    \label{fig:along_side_detect} % 图片的标签，用于在正文中引用
\end{figure}

\begin{figure}[!htp]
    \centering
    \includegraphics[]{mypic/detect_along_edge.pdf} % 大括号中为图片路径，中括号中可以添加图片的 height、weight 等大小参数，注意这里使用的是 pdf！！！
    \bicaption{沿边缘探测}{detection along the edge} % 图片的标题
    \label{fig:along_side_detect} % 图片的标签，用于在正文中引用
\end{figure}
```

例如 `图 ~\ref{fig:along_side_detect}` 编译后就会显示 `图 x-x`，其中 `x-x` 即图片的编号。

## 插入公式

Latex 中插入公式比较方便，以下代码便可以在 Latex 中插入两个公式：

```tex
\begin{equation}
    m = \lfloor \frac{X_{max}-X_{min}+W}{L} \rfloor + 1,
    \label{eq:m_cell}
\end{equation}
\begin{equation}
    n = \lfloor \frac{Y_{max}-Y_{min}+W}{L} \rfloor + 1.
    \label{eq:n_cell} % eq:n_cell 是公式的标签，用于在正文中引用，用法类似图片的 label
\end{equation}
```

$\lim\limits_{x = 0}$

## 参考

[Latex 编译（Miktex 与 texlive 异同）](https://zhuanlan.zhihu.com/p/104464775)

[SJTUThesis 说明](https://github.com/sjtug/SJTUThesis/wiki)


