---
title: Hibernate SQL查询addScalar()和addEntity()区别
date: 2016-11-02 14:57:08
comments: true
categories:
-  编程
tags:
- Hibernate
---

Hibernate除了支持HQL查询外，还支持原生SQL查询。对原生SQL查询执行的控制是通过SQLQuery接口进行的，通过执行Session.createSQLQuery()获取这个接口。该接口是Query接口的子接口。

<!-- more -->

## SQL查询步骤

1. 获取Hibernate Session对象
2. 编写SQL语句
3. 通过Session的createSQLQuery方法创建查询对象
4. 调用SQLQuery对象的 **addScalar()** 或 **addEntity()** 方法将选出的结果与标量值或实体进行关联，分别用于进行标量查询或实体查询
5. 如果SQL语句包含参数，调用Query的setXxxx方法为参数赋值
6. 调用Query的list方法返回查询的结果集


## 标量查询

最基本的SQL查询就是获得一个标量的列表：

```java
session.createSQLQuery("select * from person_inf").list();   
session.createSQLQuery("select id,name,age from person_inf").list();
```

* 它们都将返回一个**Object数组组成的List**，数组每个元素都是person_inf表的一个字段值。Hibernate会使用**ResultSetMetadata**来判定返回的标量值的实际顺序和类型。

* 但是在JDBC中过多的使用ResultSetMetadata会降低程序的性能。所以为了过多的避免使用ResultSetMetadata或者为了指定更加明确的返回值类型，我们可以使用addScalar()方法：

```java
session.createSQLQuery("select * from person_inf")    
.addScalar("name",StandardBasicTypes.STRING)  
.addScalar("age",StandardBasicTypes.INT)  
.list();
```

这个查询指定了：

1. SQL查询字符串。
2. 要返回的字段和类型。

它仍然会返回Object数组,但是此时不再使用**ResultSetMetdata**,而是明确的将name和age按照String和int类型从resultset中取出。同时，也指明了就算query是使用*来查询的，可能获得超过列出的这三个字段，也仅仅会返回这三个字段。如果仅仅只需要选出某个字段的值，而不需要明确指定该字段的数据类型，则可以使用addScalar(String columnAlias)。

实例如下：

```java
public void scalarQuery(){  
        Session session = HibernateUtil.getSession();  
        Transaction tx = session.beginTransaction();  
        String sql = "select * from person_inf";  
        List list = session.createSQLQuery(sql).  
                    addScalar("person_id",StandardBasicTypes.INTEGER).  
                    addScalar("name", StandardBasicTypes.STRING).  
                    addScalar("age",StandardBasicTypes.INTEGER).list();  
        for(Iterator iterator = list.iterator();iterator.hasNext();){  
            //每个集合元素都是一个数组，数组元素师person_id,person_name,person_age三列值  
            Object[] objects = (Object[]) iterator.next();  
            System.out.println("id="+objects[0]);  
            System.out.println("name="+objects[1]);  
            System.out.println("age="+objects[2]);  
            System.out.println("----------------------------");  
        }  
        tx.commit();  
        session.close();  
    }
```

从上面可以看出。标量查询中addScalar()方法有两个作用：

1. 指定查询结果包含哪些数据列---没有被addScalar选出的列将不会包含在查询结果中。
2. 指定查询结果中数据列的数据类型。

## 实体查询

上面的标量查询返回的标量结果集，也就是从resultset中返回的“裸”数据。如果我们想要的结果是某个对象的实体，这是就可以通过addEntity()方法来实现。addEntity()方法可以讲结果转换为实体。但是在转换的过程中要注意几个问题：

1. 查询返回的是某个数据表的全部数据列
2. 该数据表有对应的持久化类映射

这时才可以通过addEntity()方法将查询结果转换成实体。

```java
session.createSQLQuery("select * from perons_inf").addEntity(Person.class).list;    
session.createSQLQuery("select id,name,age from person_inf").addEntity(Person.class).list();
```

这个查询指定：

1. SQL查询字符串
2. 要返回的实体

假设Person被映射为拥有id,name和age三个字段的类，以上的两个查询都返回一个List，每个元素都是一个Person实体。

假若实体在映射时有一个many-to-one的关联指向另外一个实体，在查询时必须也返回那个实体（获取映射的外键列），否则会导致发生一个"column not found"的数据库错误。这些附加的字段可以使用*标注来自动返回，但我们希望还是明确指明，看下面这个具有指向teacher的many-to-one的例子：

```java
session.createSQLQuery("select id, name, age, teacherID from person_inf").addEntity(Person.class).list();
```

这样就可以通过person.getTeacher()获得teacher了。

实例：

```java
public void entityQuery(){  
    Session session = HibernateUtil.getSession();  
    Transaction tx = session.beginTransaction();  
    String sql = "select * from person_inf";  
    List list = session.createSQLQuery(sql).  
    addEntity(Person.class).    //指定将查询的记录行转换成Person实体  
     list();       
    for (Iterator iterator = list.iterator();iterator.hasNext();) {  
        Person person = (Person) iterator.next();//集合的每个元素都是一个Person对象  
        System.out.println("name="+person.getName());  
       System.out.println("age="+person.getAge());  
  	}  
    tx.commit();  
    session.close();  
}
```

上面的都是单表查询，如果我们在SQL语句中使用了多表连接，则SQL语句可以选出多个数据表的数据。Hibernate支持将查询结果转换成多个实体。如果要将查询结果转换成多个实体，则SQL字符串中应该为不同数据表指定不同别名，并且调用addEntity()方法将不同数据表转换成不同实体。如下

```java
public void multiEntityQuery(){  
    Session session = HibernateUtil.getSession();  
    Transaction tx = session.beginTransaction();  
    String sql = "select p.*,e.* from person_inf as p inner join event_inf as e" +  
                 " on p.person_id=e.person_id";  
    List list = session.createSQLQuery(sql)  
                .addEntity("p",Person.class)  
                .addEntity("e", MyEvent.class)  
                .list();  
    for(Iterator iterator = list.iterator();iterator.hasNext();){  
        //每个集合元素都是Person、MyEvent所组成的数组  
        Object[] objects = (Object[]) iterator.next();  
        Person person = (Person) objects[0];  
        MyEvent event = (MyEvent) objects[1];  
        System.out.println("person_id="+person.getId()+" person_name="+person.getName()+" title="+event.getTitle());        
    }  
}
```

>转载自：http://blog.csdn.net/vacblog/article/details/7769976
