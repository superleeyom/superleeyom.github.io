---
title: MongoDB查询缓慢问题处理
date: 2020-05-22 21:17:25
tags:
- MongoDB
categories:
- 编程
	
---



在应用系统中，尤其在联机事务处理系统中，对数据查询及处理的速度已成为衡量应用系统成败的标准。而采用索引来加快数据处理速度也成为广大数据库用户所接受的优化方法。在良好的数据库设计基础上，有效地使用索引是取得高性能的基础。最近就在实际的开发中遇到了关于错误使用索引，导致请求访问超时的问题，这里记录下。



<!--more-->

## 背景
因为业务需要，需要将 一张表的数据由 MySQL 迁移到 MongoDB，迁移完毕后，出现了外部调用接口的时候出现了大面积的超时，并且 MongoDB 所在的机器 cpu 被打到100%。
## 几种常见的索引
解决问题之前需要了解 MongoDB 的里面常见的几种索引：

- **基础索引**：MongoDB基础索引的概念跟其他数据库产品一样，即在表上为某一列建立索引，并指定表数据是按这一列升序排列还是降序排列，比如我在 out_message 表上的 serviceNo 字段加上索引： `db.out_message.ensureIndex({serviceNo:1})` ，serviceNo 后面的“1”说明表数据是按age列升序排列还是降序排列。“1”为升序排列，“-1”为降序排列。
- **文档索引**：索引不仅可以建立在普通字段上，还可以建立在嵌入式文档类型的字段上。例如，数据库中有一个factories表，这张表的addr字段是一个嵌入式文档类型。希望在addr字段上建立索引，可以使用下面的代码：
```sql
// 插入数据
db.factories.insert({ name: "wwl", addr: { city: "Beijing", state: "BJ" } } );
// 创建文档索引
db.factories.ensureIndex( { addr : 1 } );
// 走索引
db.factories.find({ addr: { city: "Beijing", state: "BJ" } } );
// 走索引
db.factories.find( { addr: { state: "BJ" , city: "Beijing"} } );
```
第一个 find，在addr列上的查询顺序跟此列的嵌入式字段定义顺序相同，即先city再state，所以这个查询会用到索引；第二个 find，应用查询命令  `db.factories.find( { addr: { state: "BJ" , city: "Beijing"} } )`  检索数据，此时在addr列上的查询顺序跟此列的嵌入式字段定义顺序不一致，有书上说无法使用索引，但是实际实验也会使用索引；

- **组合索引**：当索引包含多个列时，称为组合索引或复合索引。组合索引的好处是，假如查询语句需要返回的数据字段都在组合索引的索引字段中，数据库将不会访问数据页，而直接返回索引中的字段值，以加快查询速度。假如查询语句不满足这个条件，组合索引就是没有意义的，反而浪费存储空间。例如如下代码所示：
```sql
// 创建索引
db.factories.ensureIndex( { "addr.city" : 1, "addr.state" : 1 } )
// 走索引
db.factories.find( { "addr.city" : "Beijing", "addr.state" : "BJ" } );
// 走索引
db.factories.find( { "addr.city" : "Beijing" } ）; 
// 不走索引                  
db.factories.find().sort（ { "addr.state" : 1, "addr.city" : 1 } );
// 走索引
db.factories.find({  "addr.state" : "BJ","addr.city" : "Beijing" } );
```
我在factories表的addr.city和addr.state列上建立组合索引，然后执行第一个 find 查询数据，由于查询条件中字段的顺序与组合索引定义的字段顺序相同，所以此查询会使用到索引，第二个 find，由于查询条件中只指定了addr.city列，而且这个字段的定义也是factories表的组合索引的第1个字段，所以此查询会使用到索引，第三个 find比较特殊，通过实际实验，sort 中由于查询条件中先使用addr.state，后使用addr.city，与factories表字段顺序的定义不一致，所以此查询不会使用索引，第四个 find，在 find 中索引字段的顺序调换了一下，此查询还是会使用索引的，所以总结：

> 1. 建立复合索引，索引中两字段前后顺序与查询条件字段在数量一致的情况下，顺序不影响使用索引查询；
>
> 2. 当复合索引中的字段数量与查询条件字段数量不一致情况下，以复合索引字段前置优先规则；

- **唯一索引**：这个很容易理解，跟 MySQL 数据库里面的唯一索引一个意思，如果索引声明为唯一的，就不允许出现多个索引值相同的行。唯一索引可以确保只有两行数据里所有索引字段的值都相同的时候，插入数据时才被拒绝，即保证索引字段唯一性。比如可以使用 `db.t4.ensureIndex（{firstname: 1, lastname: 1}, {unique: true}）` 命令创建唯一索引。



## 如何解决
当时是将 MySQL 的数据迁移到 MongoDB 的 out_message 表上，数据量其实不大，大概在 58w 的数据，但是发现大量的查询超时，每个查询的接口耗时 10s 以上。起初我在 out_message 这个表上，加了两个基础索引，分别是 providerId 和 serviceNo，这两个字段用于下游的安装商拉取 MongoDB 里的工单数据用的，出现超时后，分析对应的查询语句：
```sql
db.getCollection('out_message').find({ "providerId" : "530",  "type" : "SERVICE", 
"syncState" : "Y", "_class" : { "$in" : ["com.lumi.retail.base.common.entity.ticket
.OutMessage"] } }).sort({"_id": -1}).limit(100).explain()
```
外部查询条件： `request = /providerApi/connectTaskList?providerId=455&pageSize=60&type=EXCEPTION&pageNum=1&status=N` ，发现其实是有命中 providerId 索引，indexBounds表示使用的索引，这里只截取部分：
```json
"indexBounds" : {
  "providerId" : [ 
    "[\"530\", \"530\"]"
  ]
}
```
虽然 providerId 是命中索引了，但是命中的数据超过了30多万，每次都加载这么多数据，查询起来肯定是非常慢的。所以根据用户的查询条件，创建一个组合索引和一个基础索引：

- 组合索引 idx_providerid_type_syncstate：
```json
{
    "providerId" : 1,
    "type" : 1,
    "syncState" : 1
}
```

- 基础索引serviceNo：
```json
{
    "serviceNo" : 1
}
```
因为 serviceNo 这个字段是非必填项，所以给他单独设置一个索引，而 providerId、type、syncState是查询必填字段，组成一个组合索引，那么在查询的时候，能很大几率命中复合索引，查询到的数据范围也进一步的缩小，提高了查询的效率，再次访问服务，就基本上已经正常了，目前测试环境（120w 数据）接口响应时间维持在10-50毫秒左右。
```json
"indexBounds" : {
  "providerId" : [ 
    "[\"413\", \"413\"]"
  ],
  "type" : [ 
    "[\"SERVICE\", \"SERVICE\"]"
  ],
  "syncState" : [ 
    "[\"Y\", \"Y\"]"
  ]
}
```


## 总结
对于MongoDB来说，如果查询语句的执行计划中nscanned（扫描的记录数）远大于nreturned（返回结果的记录数），那么就要考虑通过创建索引来优化记录定位了。对于创建索引的建议：如果很少读，那么尽量不要添加索引，因为索引越多，插入和更新操作会越慢。如果读的操作很多，那么创建索引还是比较合适的。