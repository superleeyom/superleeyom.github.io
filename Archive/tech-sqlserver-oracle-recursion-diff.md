---
title: sqlserver、oracle、MySQL 递归查询的区别
date: 2016-11-01 22:05:04
comments: true
categories:
- 编程
tags:
- 递归查询
---

程序调用自身的编程技巧称为递归（ recursion）。递归做为一种算法在程序设计语言中广泛应用。 一个过程或函数在其定义或说明中有直接或间接调用自身的一种方法，它通常把一个大型复杂的问题层层转化为一个与原问题相似的规模较小的问题来求解，递归策略只需少量的程序就可描述出解题过程所需要的多次重复计算，大大地减少了程序的代码量。递归的能力在于用有限的语句来定义对象的无限集合。一般来说，递归需要有边界条件、递归前进段和递归返回段。当边界条件不满足时，递归前进；当边界条件满足时，递归返回。

<!--more-->

以上是摘自百度百科对于递归的定义，由于经常在项目中遇到递归查询，但是因为数据库方言的不同，三种数据库的递归查询语句都不相同，所以就做个简单的总结。

## 表结构

![2016121052968QQ20161210-095910@2x.png](http://image.leeyom.top/2016121052968QQ20161210-095910@2x.png)

```
id --主键
name --区域名称
topgrid_id --上级区域id
```

## oracle 写法

```sql
SELECT ID,NAME,TOPGRID_ID
FROM GRID START WITH id = 1715 CONNECT BY PRIOR ID = TOPGRID_ID;
```

* 关键字：**start with 。。。 connect by prior 。。。**
* 含义：查询节点为1715下面所有的子节点，包括本身。

## sqlserver 写法

```sql
WITH cte AS
(
  SELECT
    ID,
    NAME,
    TOPGRID_ID
  FROM GRID
  WHERE id = 1715
  UNION ALL
  SELECT
    g.ID,
    g.NAME,
    g.TOPGRID_ID
  FROM cte c INNER JOIN Grid g
      ON c.id = g.TOPGRID_ID
)
SELECT *
FROM cte;
```

* 关键字：**WITH 。。。 AS 。。。**

## MySQL 写法

因为MySQL中没有connect by 这种写法，也没有with子句，需要定义一个函数去帮我们实现我们的功能。


```sql
DROP FUNCTION IF EXISTS queryChildren;
CREATE FUNCTION 'queryChildren' (areaId INT)
RETURNS VARCHAR(4000)
BEGIN
DECLARE sTemp VARCHAR(4000);
DECLARE sTempChd VARCHAR(4000);

SET sTemp = '$';
SET sTempChd = cast(areaId as char);

WHILE sTempChd is not NULL DO
SET sTemp = CONCAT(sTemp,',',sTempChd);
SELECT group_concat(id) INTO sTempChd FROM grid WHERE FIND_IN_SET(parentId,sTempChd)>0;
END WHILE
return sTemp;
END;
```

在sql语句使用该函数

```sql
select * from GRID where find_in_set(id,queryChildrenAreaInfo(1715));  
```
可以将id为1715的所有下级全部查询出来
