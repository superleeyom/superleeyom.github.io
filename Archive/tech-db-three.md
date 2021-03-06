---
title: 数据库三范式
date: 2017-04-04 20:28:14
comments: true
tags:
- 数据库
categories:
- 编程
---
设计关系数据库时，遵从不同的规范要求，设计出合理的关系型数据库，这些不同的规范要求被称为不同的范式，各种范式呈递次规范，越高的范式数据库冗余越小。如何去理解数据库的三范式呢？下面是根据知乎上面的一些解答，总结出来的。

<!--more-->

## 第一范式

**属性不可分割**。所谓的属性，就是我们说的字段，就比如学生信息表由姓名、年龄、性别、学号等组成，不可分割的意思按照字面理解就是不能拆分成最小的单位。但是在国外，姓名是要分开的，也就是说姓名这个字段可以在分为 first name 和 last name，那么我们设计的这个表是不符合第一范式的。

## 第二范式

第二范式就是 **主键依赖**，其他的字段都依赖主键。通过主键，我就可以获取到其他字段的值。如果不依赖主键，我们就找不到他们。就像为什么学生信息表里面姓名不可以做主键，因为姓名存在同名，就违背了第二范式，学号是唯一的，是可以做为主键。

## 第三范式

第三范式就是要 **消除传递依赖**，通俗一点就是消除冗余。消除冗余应该比较好理解一些，就是各种信息只在一个地方存储，不出现在多张表中。就好比如说一个大学分了很多的系（计算机系、中文系、英语系。。。），这个系别管理表信息有以下字段组成：系编号，系主任，系简介，系架构。学生信息表中已经有姓名、年龄、学号、性别等字段，那能不能把系编号、系主任、系简介等字段一起存在学生信息表中呢？因为我们已经有系别管理表去存放系相关信息，再放到学生信息表中，就出现冗余，也就是传递依赖。这样就不符合第三范式，那么正确的做法是我们只需要添加一个系编号字段就行，根据该系编号字段，就可以获取到具体的系别信息，这就是第三范式。

## 特殊说明

字段允许适当冗余，以提高查询性能，但必须考虑数据一致。冗余字段应遵循：
- 不是频繁修改的字段。
- 不是 varchar 超长字段，更不能是 text 字段。

例如：商品类目名称使用频率高，字段长度短，名称基本一成不变，可在相关联的表中冗余存储类目名称，避免关联查询。
