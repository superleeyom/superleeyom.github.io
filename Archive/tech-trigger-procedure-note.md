---
title: 触发器与存储过程讲义
date: 2017-04-27 10:54:56
tags:
- 触发器
comments: true
categories:
- 编程
---

什么是触发器？什么是存储过程？因为用的少，对这两块的知识有点模糊，趁着公司每周三技术分享，通过网上资料和看书，将这两块的知识点梳理了下，做成了一个讲义，顺便就放到博客上来了，随便看看就好，因为如果不是专业的DBA，平常也很难涉及到触发器和存储过程，而且在阿里巴巴的java开发手册中，明确的表示，尽量避免使用存储过程，所以，看看就好。

<!--more-->

# TRIGGER(触发器)

## 1. 什么是触发器?
触发器，从字面来理解，一触即发的一个器，简称触发器（哈哈，个人理解），举个例子吧，好比天黑了，你开灯了，你看到东西了。你放炮仗，点燃了，一会就炸了。在MySQL Server里面也就是对某一个表的一定的操作，触发某种条件（Insert,Update,Delete 等），从而自动执行的一段程序，这就是触发器。

支持触发器的语句有:

>DELETE;

>INSERT;

>UPDATE;

其他的mysql语句不支持触发器，像select语句等。

## 2. 如何创建触发器？

在创建触发器的时候，需要给出4条信息：

1. 唯一的触发器名；
2. 触发器关联的表；
3. 触发器应该响应的活动(DELETE、INSERT、UPDATE);
4. 触发器何时执行(处理之前还是之后)；

> 在MySQL5中，触发器名必须在每个表中唯一，但不是在每个数据库中唯一。这表示同一数据库中的两个表可具有相同名字的触发器。

示例:

```sql
CREATE TRIGGER newproduct
AFTER INSERT ON products
FOR EACH ROW INSERT INTO message(message) VALUES('Product added');
```

分析:

`CREATE TRIGGER` 用来创建名为 `newproduct` 的新触发器。触发器可在一个操作发生之前或之后执行，这里给出了 `AFTER INSERT`，所以此触发器将在 `INSERT` 语句成功执行后执行。这个触发器还指定 `FOR EACH ROW`，因此代码对每个插入行执行。在这个例子中， 将每次products插入一行的时候，同时往message的表中插入文本 `Product added`。

注意：

1. 只有表才支持触发器，视图不支持，临时表也不支持。
2. 每个表最多支持6个触发器（每条INSERT、UPDATE 和DELETE的之前和之后）

## 3 .如何删除触发器？

示例：

```sql
DROP TRIGGER newproduct
```

注意：触发器不能更新或者覆盖，为了修改一个触发器，必须先删除它，再重新创建

## 4. 如何使用触发器？

### 4.1 NSERT触发器

INSERT触发器就是当对定义触发器的表执行INSERT语句时，就会调用的触发器，INSERT触发器可以用来修改，甚至拒绝接受正插入的记录。

下面来看一个实例：

首先先创建ClassInfo(班级表)、StudentInfo(学生表)

教室表

```sql
DROP TABLE IF EXISTS `classInfo`;
CREATE TABLE `classInfo` (
  `ClassNo` varchar(45) NOT NULL,
  `ClassName` varchar(45) DEFAULT NULL,
  `TotalNum` varchar(45) DEFAULT NULL,
  `TeacherName` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`ClassNo`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

BEGIN;
INSERT INTO `classInfo` VALUES ('002', '计算机一班', '4', '王老师');
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;
```

学生表

```sql
DROP TABLE IF EXISTS `StudentInfo`;
CREATE TABLE `StudentInfo` (
  `StuName` varchar(45) DEFAULT NULL,
  `StuNo` varchar(45) NOT NULL,
  `StuClass` varchar(45) DEFAULT NULL,
  `Sex` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`StuNo`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```


创建触发器

```sql
CREATE TRIGGER T_addStudent

AFTER INSERT

  ON StudentInfo FOR EACH ROW

  UPDATE classInfo

  SET TotalNum = TotalNum + 1

  WHERE ClassNo = (SELECT DISTINCT ClassNo

                   FROM StudentInfo);

```

上面这段代码就是建立了一个插入触发器

![](https://ww1.sinaimg.cn/large/006tNbRwly1fex17cloyrj30vm0c2myw.jpg)

如上图所示

1. 写明触发器的名称
2. 该触发器是在那个表改变的时候发生
3. 当这个表进行什么操作的时候发生
4. 发生上述操作之后还要进行怎样的操作。

这段代码的意思是：当在studentInfo表中添加一条记录的时候，就要更新ClassInfo中的TotalNum这一列，这一列的数据要增加1

下面我们验证一下：
输入下面的代码：

```sql
SELECT totalNum
FROM ClassInfo
WHERE ClassNo = '002';

INSERT INTO StudentInfo VALUES ('小明', '003', '002', '男');
INSERT INTO StudentInfo VALUES ('小花', '002', '002', '女');
INSERT INTO StudentInfo VALUES ('小高', '004', '002', '女');

SELECT totalNum
FROM ClassInfo
WHERE ClassNo = '002';

```

### 4.2 DELETE触发器

当数据库运行DELETE语句时，就会激活DELETE触发器，DELETE触发器用于约束用户能够从数据库中删除的数据，因为这些数据中，有些数据是不希望用户轻易删除的。

接下来我们创建一个TeacherInfoFor(老师信息)表，如下：

```sql
DROP TABLE IF EXISTS `TeacherInfoFor`;
CREATE TABLE `TeacherInfoFor` (
  `TeacherID` varchar(45) NOT NULL,
  `TeacherName` varchar(45) DEFAULT NULL,
  `Sex` varchar(45) DEFAULT NULL,
  `Age` int(45) DEFAULT NULL,
  `Telephone` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`TeacherID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

BEGIN;
INSERT INTO `TeacherInfoFor` VALUES
('001', '王芳', '女', '23', '158'),
('003', '张丽', '女', '28', '152'),
('004', '张明', '男', '30', '138');
COMMIT;
```

创建Stu_Teacher(学生教师表)，如下：

```sql
DROP TABLE IF EXISTS `Stu_Teacher`;
CREATE TABLE `Stu_Teacher` (
  `StuNo` varchar(45) NOT NULL,
  `TeacherID` varchar(45) DEFAULT NULL,
  `StuName` varchar(45) DEFAULT NULL,
  `ID` int(45) NOT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

BEGIN;
INSERT INTO `Stu_Teacher` VALUES
('001', '004', '二丫', '1'),
('002', '004', '王小二', '2'),
('003', '004', '花花', '3');
COMMIT;
SET FOREIGN_KEY_CHECKS = 1;
```

创建触发器：

```sql
CREATE TRIGGER T_DELETETEACHERon
AFTER DELETE ON TeacherInfoFor
FOR EACH ROW DELETE FROM Stu_Teacher
WHERE TeacherID = old.TeacherID;
```
该触发器的作用是，当删除某个教师的信息的时候，这个老师下面的学生信息也将一并的删除掉。

测试：

```sql
DELETE FROM TeacherInfoFor WHERE TeacherID = '004';
```

### 4.3 UPDATE触发器

当一个UPDATE语句在目标表上运行的时候，就会调用UPDATE触发器，这种类型的触发器专门用于约束用户能修改的现有的数据。
这个时候举个例子，先准备数据环境：

```sql
-- 创建表shop_product
DROP TABLE IF EXISTS `shop_product`;
CREATE TABLE `shop_product` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `on_sale_time` date DEFAULT NULL,
  `book_name` varchar(45) COLLATE utf8_bin DEFAULT NULL,
  `on_sale` int(45) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `shop_product_id_uindex` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

-- 往表中插入一条数据
BEGIN;
INSERT INTO `shop_product` VALUES ('1', null, 'java编程思想', '0');
COMMIT;
```

表（shop\_product ）中有一状态值--是否上架(on\_sale) 若由未上架（0）转为上架（1） 同时设置上架时间（on\_sale\_time）

触发器写法如下:

```sql
DROP TRIGGER IF EXISTS update_on_sale_time_of_product;
CREATE TRIGGER update_on_sale_time_of_product BEFORE UPDATE ON shop_product
FOR EACH ROW
  BEGIN
    IF OLD.on_sale = 0 && NEW.on_sale = 1
    THEN
      SET NEW.on_sale_time = now();
    END IF;
  END;
```

测试：

```sql
UPDATE shop_product SET on_sale = 1 WHERE id = 1;
```

最终结果 on\_sale\_time 字段更新为该商品上架时间!


## 5. 总结

1. 明确触发器语法的四要素：

 - 监视地点(table)
 - 监视事件(insert/update/delete)
 - 触发时间(after/before)
 - 触发事件(insert/update/delete)

2. 触发器语法

 ```sql
 create trigger triggerName  
after/before insert/update/delete on 表名  
for each row   
begin  
        sql语句;  
end;
 ```

3. 个人理解

 我的一般理解就是触发器是一个隐藏的存储过程，因为它不需要参数，不需要显示调用，往往在你不知情的情况下已经做了很多操作。

# PROCEDURE(存储过程)

## 1. 什么是存储过程？

存储过程简单来说，就是为以后的使用而保存的一条或多条MySQL语句的集合。可将其视为批件，虽然它们的作用不仅限于批处理。
在我看来， <span style="color:red;">存储过程就是有业务逻辑和流程的集合</span>， 可以在存储过程中创建表，更新数据， 删除等等。

## 2. 为什么要使用存储过程？

1. 通过把处理封装在容易使用的单元中，简化复杂的操作（类似于java里面的封装性）。
2. 由于不要求反复建立一系列处理步骤，这保证了数据的完整性。如果所有开发人员和应用程序都使用同一（试验和测试）存储过程，则所使用的代码都是相同的。这一点的延伸就是防止错误。需要执行的步骤越多，出错的可能性就越大。防止错误保证了数据的一致性。
3. 简化对变动的管理。如果表名、列名或业务逻辑（或别的内容）有变化，只需要更改存储过程的代码。使用它的人员甚至不需要知道这些变化。

## 3. 一个简单的存储过程

```sql
create procedure porcedureName ()
begin
    select StuName from Stu_Teacher;
end;
```

存储过程用create procedure 创建， 业务逻辑和sql写在begin和end之间。mysql中可用call porcedureName ();来调用过程。

```sql
- 调用过程
call porcedureName ();
```

该存储过程没有参数， 只是在调用的时候查询了Stu\_Teacher表的用户名而已。

## 4. 删除存储过程

```sql
DROP PROCEDURE IF EXISTS porcedureName; -- 没有括号()
```

## 5. 使用参数的存储过程

```sql
create procedure procedureName(
    out min decimal(8,2),
    out avg decimal(8,2),
    out max decimal(8,2)
)
BEGIN
    select MIN(Age) INTO min from TeacherInfoFor;
    select AVG(Age) into avg from TeacherInfoFor;
    select MAX(Age) into max from TeacherInfoFor;
END;
```

此过程接受三个参数， 分别用于获取TeacherInfoFor表的最小、平均、最大年龄。每个参数必须具有指定的类
型，这里使用十进制值（decimal(8,2)）， 关键字<span style="color:red;">OUT</span>指出相应的参数用来从存储过程传出
一个值（返回给调用者）。

> MySQL支持IN（传递给存储过程）、OUT（从存储过程传出，如这里所用）和INOUT（对存储过程传入和传出）类型的参数。存储过程的代码位于BEGIN和END语句内，如前所见，它们是一系列SELECT语句，用来检索值，然后保存到相应的变量（通过指定INTO关键字）

为调用此修改过的存储过程，必须指定3个变量名，如下所示：(所有MySQL变量都必须以@开始。)

```sql
-- 由于过程指定三个参数， 故调用必须要参数匹配
call procedureName(@min, @avg, @max);
```

该调用并没有任何输出， 只是把调用的结果赋给了调用时传入的变量（@min, @avg, @max）。然后即可调用显示该变量的值。

```sql
select @min, @avg, @max;
```

使用in参数, 输入一个老师的id， 返回该老师下面的所有学生人数总和。

```sql
create procedure getTotalById (
    in teacher_id VARCHAR(45),
    out total int
)
BEGIN
    select count(s.StuNo) from Stu_Teacher s
    where s.TeacherID = teacher_id
    into total;
END;
```

调用存储过程

```sql
call getTotalById('004', @total);
select @total;
```

## 6. 为什么不推荐使用Mysql触发器而用存储过程？

1. 存储过程和触发器二者是有很大的联系的，我的一般理解就是触发器是一个隐藏的存储过程，因为它不需要参数，不需要显示调用，往往在你不知情的情况下已经做了很多操作。从这个角度来说，**由于是隐藏的，无形中增加了系统的复杂性，非DBA人员理解起来数据库就会有困难，因为它不执行根本感觉不到它的存在**。
2. 再有，涉及到复杂的逻辑的时候，触发器的嵌套是避免不了的，如果再涉及几个存储过程，再加上事务等等，很容易出现死锁现象，再调试的时候也会经常性的从一个触发器转到另外一个，级联关系的不断追溯，很容易使人头大。其实，从性能上，触发器并没有提升多少性能，只是从代码上来说，可能在coding的时候很容易实现业务，所以我的观点是：**摒弃触发器！触发器的功能基本都可以用存储过程来实现。**
3. 在编码中存储过程显示调用很容易阅读代码，触发器隐式调用容易被忽略。
存储过程也有他的致命伤↓
4. 存储过程的致命伤在于移植性，存储过程不能跨库移植，比如事先是在mysql数据库的存储过程，考虑性能要移植到oracle上面那么所有的存储过程都需要被重写一遍。

# 参考资料

- 《MySQL必知必会》: 链接: http://pan.baidu.com/s/1kVoNfGN 密码: 9acp
- https://segmentfault.com/a/1190000006756268
