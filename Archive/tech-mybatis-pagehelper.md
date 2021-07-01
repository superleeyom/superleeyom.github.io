---
title: mybatis 分页插件PageHelper使用及总结
date: 2016-12-05 22:27:09
comments: true
categories:
- 编程
tags:
- PageHelper
---


今天在看视频教程的时候，遇到了mybatis的分页插件`PaheHelper`，感觉很好用，它支持主流的数据库，该插件目前支持`Oracle`,`Mysql`,`MariaDB`,`SQLite`,`Hsqldb`,`PostgreSQL`六种数据库分页。

<!--more-->

## 原理
![2016120556503mybatis分页查询原理.png](http://og1m51u2s.bkt.clouddn.com/2016120556503mybatis分页查询原理.png)

## 思路

1. 在SqlMapConfig.xml中配置plugin
2. 在sql语句查询之前，<span style="color:red;">**PageHelper.startPage(page, rows)**</span>
3. 获取分页结果，<span style="color:red;">**PageInfo<TbItem> info = new PageInfo<>(list);**</span>list参数是查询到的结果集，pageInfo封装了分页信息

## 实现步骤

### 添加依赖

在pom.xml中添加如下的依赖:

```xml
<dependency>
	<groupId>com.github.pagehelper</groupId>
	<artifactId>pagehelper</artifactId>
	<version>${pagehelper.version}</version>
</dependency>
```

### 修改SqlMapConfig.xml

在mubatis配置文件中添加插件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
		PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
		"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<!-- 配置分页插件 -->
	<plugins>
		<plugin interceptor="com.github.pagehelper.PageHelper">
			<!-- 指定数据库方言 -->
			<property name="dialect" value="mysql"/>
		</plugin>
	</plugins>
</configuration>
```

### 前台
前台采用easyUI的dataGrid组件：

```html
<table class="easyui-datagrid" id="itemList" title="商品列表"
       data-options="singleSelect:false,collapsible:true,pagination:true,url:'/item/list',method:'get',pageSize:30,toolbar:toolbar">
    <thead>
        <tr>
        	<th data-options="field:'ck',checkbox:true"></th>
        	<th data-options="field:'id',width:60">商品ID</th>
            <th data-options="field:'title',width:200">商品标题</th>
            <th data-options="field:'cid',width:100">叶子类目</th>
            <th data-options="field:'sellPoint',width:100">卖点</th>
            <th data-options="field:'price',width:70,align:'right',formatter:TAOTAO.formatPrice">价格</th>
            <th data-options="field:'num',width:70,align:'right'">库存数量</th>
            <th data-options="field:'barcode',width:100">条形码</th>
            <th data-options="field:'status',width:60,align:'center',formatter:TAOTAO.formatItemStatus">状态</th>
            <th data-options="field:'created',width:130,align:'center',formatter:TAOTAO.formatDateTime">创建日期</th>
            <th data-options="field:'updated',width:130,align:'center',formatter:TAOTAO.formatDateTime">更新日期</th>
        </tr>
    </thead>
</table>
	```

### 后台

#### pojo实体

我们需要一个实体去封装我们获取到分页信息，这个实体被解析为json字符串，必须符合dataGrid的数据格式：

```java
package com.taotao.pojo;
import java.util.List;
public class EasyUIDataGridResult {

	/**
	 * easyUI dataGrid 返回结果封装
	 */
	// 总的记录数
	Long total;
	// 数据集
	List<?> rows;

	public Long getTotal() {
		return total;
	}

	public void setTotal(Long total) {
		this.total = total;
	}

	public List<?> getRows() {
		return rows;
	}

	public void setRows(List<?> rows) {
		this.rows = rows;
	}
}

```

#### dao层

```java
List<TbItem> selectByExample(TbItemExample example);
```

#### service层

```java
@Override
	public EasyUIDataGridResult getItemList(int page, int rows) {

		//分页处理
		PageHelper.startPage(page, rows);

		//查询结果
		TbItemExample example = new TbItemExample();
		List<TbItem> list = itemMapper.selectByExample(example);

		//获取分页信息
		PageInfo<TbItem> info = new PageInfo<>(list);
		EasyUIDataGridResult result = new EasyUIDataGridResult();
		long total = info.getTotal();

		//封装分页信息
		List<TbItem> row = info.getList();
		result.setRows(row);
		result.setTotal(total);
		return result;
	}

```

#### controller层

```java
/**
	 * 商品列表，分页
	 * @param page
	 * @param rows
	 * @return
	 */
	@RequestMapping("/item/list")
	@ResponseBody
	private EasyUIDataGridResult getItemList(Integer page,Integer rows){
		return itemService.getItemList(page, rows);
	}
```

## 效果图
![2016120579015QQ20161205-1@2x.png](http://og1m51u2s.bkt.clouddn.com/2016120579015QQ20161205-1@2x.png)

## 总结

pageHelper会使用ThreadLocal获取到同一线程中的变量信息，**各个线程之间的Threadlocal不会相互干扰**，也就是Thread1中的ThreadLocal1之后获取到Tread1中的变量的信息，不会获取到Thread2中的信息。
所以在多线程环境下，各个Threadlocal之间相互隔离，可以实现，不同thread使用不同的数据源或不同的Thread中执行不同的SQL语句。所以，**PageHelper利用这一点通过拦截器获取到同一线程中的预编译好的SQL语句之后将SQL语句包装成具有分页功能的SQL语句，并将其再次赋值给下一步操作，所以实际执行的SQL语句就是有了分页功能的SQL语句**

>参考：http://blog.csdn.net/jaryle/article/details/52315565
