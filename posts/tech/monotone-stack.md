---
title: "monotone stack"
date: 2022-11-11T15:55:04+08:00
lastmod: 2022-11-11T15:55:04+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["data structure and algorithms", "monotone stack"]
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
## Description
### brief
Monotone stack is a stack whose elements(from top to bottom) are (strictly) monotonically increasing or decreasing.

*Monotone increasing stack*: the element which is smaller than the element in the top can be pushed into stack, else we will pop the element in the top, until the stack is empty or the element is smaller than the element in the top, then we push the element into the stack. This data structure is usually used for problems to find first element that is larger than certain element.

*Monotone decreasing stack*: is the opposite of a monotonically increasing stack, used for problems to find first element that is smaller than certain element.

### how to judge
Monotonically increasing/decreasing stacks are generally determined by the order in which they exit the stack

### the sentry tips
Sometimes, if all the elements in the array is needed, that is: all the elements in the stack will be popped. It's very likely that the border will not be passed because it is not considered. So we can use the *sentinel method*.

For example, adding a `-1` at the end of {1, 3, 4, 5, 2, 9, 6} as a sentinel, it becomes {1, 3, 4, 5, 2, 9, 6, -1}, a trick that simplifies the code logic.

## Examples

