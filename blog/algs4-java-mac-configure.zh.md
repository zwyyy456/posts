---
title: "macOS 配置算法（第四版）的开发环境"
date: 2023-06-22T17:36:50+08:00
lastmod: 2023-06-22T17:36:50+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["java", "trick"]
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
## Java 环境配置

前往 [Adoptium](https://adoptium.net/zh-CN/) 下载他们预编译的 JDK 17（最新的 LTS 版本）的安装器，安装好之后，命令行执行 `java -version`，输出如下：

```sh
openjdk version "17.0.7" 2023-04-18
OpenJDK Runtime Environment Temurin-17.0.7+7 (build 17.0.7+7)
OpenJDK 64-Bit Server VM Temurin-17.0.7+7 (build 17.0.7+7, mixed mode)
``` 

说明环境变量已经自动配置好了。

同时前往 Jetbrains 官网下载 IntelliJ IDEA CE（懒得再申请教育优惠了），安装好之后打开，在 `~/Documets/zCode/Algs_4th/` 目录下创建名为 algs4 的新项目，JDK 选择我们安装的 JDK 17。如下图：

![1aEVdTMXmteJBWG](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065634.png)

## algs4 配置

先去书籍官网下载 [algs4.jar](https://algs4.cs.princeton.edu/code/algs4.jar)，我这里直接放到了上面 IDEA 创建的项目的目录下，即 `~/Documets/zCode/Algs_4th/algs4/`，然后用 IDEA 打开该项目，`File->Project Structure->Modules->Dependencies` 点击 Module SDK 下面的加号，选择 JARs or directories，再选择我们放在项目目录下的 `algs4.jar` 文件，然后就会看到 `algs4.jar` 已经被添加到该工程的 Dependencies 依赖包中，勾选，然后点击确定，就完成了环境的搭建。

![YyEHLaTOJ1WbQfX](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065636.png)

