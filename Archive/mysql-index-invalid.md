---
title: mysql 索引失效的情况
date: 2019-07-27 16:15:09
tags:
- 索引
categories:
- 编程
---

索引是对数据库表中一列或多列的值进行排序的一种结构，使用索引可快速访问数据库表中的特定信息。数据库索引本质上是一种数据结构(存储结构+算法)，目的是为了加快目标数据检索的速度。当使用关联查询（inner 、left、right  join）等进行查询时候，关联条件都已建立索引，但查看执行计划发现并未走索引，总结下实际开发中，遇到的索引失效的情况。

<!--more-->

执行如下的 sql 语句：

```mysql
EXPLAIN SELECT
	t.ticket_id,
	t.product_id ,
	cp.product_name
FROM
	ticket t
LEFT JOIN config_product cp ON t.product_id = cp.product_id
WHERE
	t.ticket_id = 1446400;
```

发现 config_product 并未用到 product_id 字段的索引，经过分析，索引失效的原因的主要原因有如下几种：

- 索引列字段的数据类型不相同
- 索引列字段的字符集不相同
- 对索引列进行运算，包括(+，-，*，/，! 等) 
- 数据库中的数据太少也会导致索引失效
  - 如果索引列所在的表数据量非常小，比如 10 条以下，索引不会生效，多加几条记录就可以生效
- 没有查询条件，或者查询条件没有建立索引
- 查询条件包含 or
- 模糊查询
- 组合索引，不是使用第一列索引，索引失效

遇到的问题是第一种，`ticket.product_id` 字段的数据类型为 bigint，而 `config_product.product_id` 字段的数据类型为 varchar，就导致索引失效，将 `ticket.product_id` 字段的数据类型改为 varchar，索引便生效。

