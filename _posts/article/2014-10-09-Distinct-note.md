---
layout: post
title: Distinct用法小记
category: article
---

这两天用到Distinct的时候，关于它的用法有那么点不确定，于是稍微多了解了一下，都是一些基础的东西。

1. Distinct的位置
   单独使用Distinct的时候，只能放在最前面，否则报错。
   与其他函数（count()）一起使用的时候则无限制。

2. 用法
   基本的用法也就是去重，但是对于`select distinct user_id, group_id from tableA`这样的查询语句，返回的是`user_id`和`group_id`同时不相同的结果，也就是说distinct同时作用于`user_id`和`group_id`。

    > `select distinct (user_id), group_id from tableA`的效果跟`select distinct user_id, group_id from tableA`是一样的

   如果只希望对`user_id`去重，可以考虑使用`group_by`。

关于Distinct的一些原理，可以看[这里](http://isky000.com/database/mysql_distinct_implement)。
