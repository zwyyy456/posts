---
title: "Cmake 基础教程"
date: 2023-04-13T13:42:31+08:00
lastmod: 2023-04-13T13:42:31+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["tutorial", "cpp"]
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
## 介绍
CMake是个一个开源的跨平台自动化建构系统，用来管理软件建置的程序，并不依赖于某特定编译器，并可支持多层目录、多个应用程序与多个库。 它用配置文件控制建构过程（build process）的方式和Unix的make相似，只是CMake的配置文件取名为`CMakeLists.txt`。CMake并不直接建构出最终的软件，而是产生标准的建构档（如Unix的`Makefile`或Windows Visual C++的`projects/workspaces`），然后再依一般的建构方式使用。

## CmakeLists.txt
一个简单的`CmakeLists.txt`示例如下:
```
# 指定最小 CMake 版本要求
cmake_minimum_required(VERSION 3.9)
# 设置项目名称
project(answer)

#[[
添加可执行文件 target，类似于原来 Makefile 的：

    answer: main.o answer.o
    main.o: main.cpp answer.hpp
    answer.o: answer.cpp answer.hpp

CMake 会自动找到依赖的头文件，因此不需要特别指定，
当头文件修改的时候，会重新编译依赖它的目标文件。
#]]
add_executable(answer main.cpp answer.cpp)

#[[
使用如下命令构建本项目：

    cmake -B build      # 生成构建目录
    cmake --build build # 执行构建
    ./build/answer      # 运行 answer 程序
#]]
```

其中`cmake -B build`命令中的`-B`参数是可选的，生成的文件会放到`build`文件夹中（没有该文件夹则会自动创建，最好原先没有）。

`cmake --build build`是执行构建，生成可执行文件，`build`指的是上一步`-B`参数指定的文件夹。

## 分离库文件情形下的CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.9)
project(answer)

# 添加 libanswer 库目标，STATIC 指定为静态库
add_library(libanswer STATIC answer.cpp)

add_executable(answer main.cpp)

# 为 answer 这一可执行目标链接库 libanswer
target_link_libraries(answer libanswer)
```

## 子目录设置
我们考虑将answer相关的头文件和源文件都放入到`answer`目录下，则在`CmakeLists.txt`中要做对应修改，使`cmake`会检索对应目录下的文件。

![目录结构](https://pic-upyun.zwyyy456.tech/smms/2023-12-26-065845.png)


```cmake
cmake_minimum_required(VERSION 3.9)
project(answer)

# 添加 answer 子目录
add_subdirectory(answer)

add_executable(answer_app main.cpp)
target_link_libraries(answer_app libanswer)

#[[
使用如下命令构建本项目：

    cmake -B build      # 生成构建目录
    cmake --build build # 执行构建
    ./build/answer_app  # 运行 answer_app 程序
#]]
```

子目录中`CmakeLists.txt`内容如下：
```cmake
add_library(libanswer STATIC answer.cpp)

#[[
message 可用于打印调试信息或错误信息，除了 STATUS
外还有 DEBUG WARNING SEND_ERROR FATAL_ERROR 等。
#]]
message(STATUS "Current source dir: ${CMAKE_CURRENT_SOURCE_DIR}")

#[[
给 libanswer 库目标添加 include 目录，PUBLIC 使
这个 include 目录能被外部使用者看到。

当链接 libanswer 库时，这里指定的 include 目录会被
自动添加到使用此库的 target 的 include 路径中。
CMAKE_CURRENT_SOURCE_DIR 表示当前 CmakeLists.txt 所在的目录；

PUBLIC 标志使得 头文件接口会被传递给依赖 libanswer 库的文件
#]]
target_include_directories(libanswer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

## 使用系统安装的第三方库，以curl为例
```cmake
#[[
find_package 用于在系统中寻找已经安装的第三方库的头文件和库文件
的位置，并创建一个名为 CURL::libcurl 的库目标，以供链接。
#]]
find_package(CURL REQUIRED)

add_library(libanswer STATIC answer.cpp)

target_include_directories(libanswer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

#[[
为 libanswer 库链接 libcurl，这里 PRIVATE 和 PUBLIC 的区别是：
CURL::libcurl 库只会被 libanswer 看到，根级别的 main.cpp 中
无法 include curl 的头文件。
#]]
target_link_libraries(libanswer PRIVATE CURL::libcurl)
```