---
title: "Cmu Database 02"
date: 2023-08-07T22:36:36+08:00
lastmod: 2023-08-07T22:36:36+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["notes"]
tags: ["cmu", "database"]
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
## SQL 的使用

`where` 可以搭配 `in` 来使用，不只是使用 `=`，例如

```sql
select round (
    sum(order_date = customer_pref_delivery_date) * 100 /
    count(*),
    2
) as immediate_percentage
from Delivery
where (customer_id, order_date) in (
    select customer_id, min(order_date)
    from delivery
    group by customer_id
)
```

```sql
select round(
    count(
        count(player_id) / count(if (d.event_date = DATEADD(l.mindate, INTERVAL 1 DAY)))
    )
) as fraction
from (
    select player_id, min(event_date) as mindate
    from delivery
    group by player_id
) l left join delivery as d
on l.player_id = d.player_id
```

## 窗口函数

窗口函数的基本语法:

`<window function> over (patition by (column) order by(column))`，window function 包括专用 window function：`rank`，`dense_rank`，`row_number` 等，以及聚合函数 `sum`、`avg`、`max`、`min` 等。

之所以叫窗口函数，是因为 partition by 分组之后的结果就称为“窗口”，表示一个“范围”。

## common t able expressions

作为 window 或者 nested queries 的替代，cte 可以认为是在当此 query 范围内有效的 tempopary table，例如:
```sql
with 
    cte1 as (select * from table1)
select 
    * 
from cte1
```