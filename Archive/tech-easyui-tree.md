---
title: easyUI tree组件的构建以及异步加载
date: 2016-10-21 14:57:08
comments: true
categories:
- 编程
tags:
- easyUI
---

在我们平常的编写的web的应用中，很多时候都会用到 tree，比如一些功能树，一些有层级关系的树等等，之前一直搞不明白这种树结构是如何编写出来的，通过在项目中的实际操作，总算有了些初步的认识，下面来好好总结下 easyUI tree 的构成。

<!--more-->

## 相关表

我要构建的这颗树所相关的表BookOrg结构如下

```sql
BOOKORGID     NUMBER           --主键                       
ORGID         NUMBER        Y                         
BOO_BOOKORGID NUMBER        Y  --上级机构的id                       
ORGCODE       VARCHAR2(500) Y                         
ORGCNNAME     VARCHAR2(200) Y  --机构中文名称                       
ORGENNAME     VARCHAR2(100) Y                         
ORGCNSHORT    VARCHAR2(50)  Y                         
ORGCNSPELL    VARCHAR2(500) Y                         
ORGENSHORT    VARCHAR2(20)  Y                         
ORGENSPELL    VARCHAR2(500) Y                         
ORGLEVEL      INTEGER       Y                         
ORGCHARGER    VARCHAR2(50)  Y                         
ORGEMAIL      VARCHAR2(100) Y                         
ORGADDRESS    VARCHAR2(200) Y  --地址                       
ORGREMARK     VARCHAR2(200) Y  --备注                       
ORGSORTID     INTEGER       Y  --机构排序id                       
ORGCANCELFLAG CHAR(1)       Y  --用于逻辑删除的字段
```

## 实体类

其实easyUI 树构建最主要的部分就是tree 的json字符串的拼接，这需要我们根据自己的业务需求在后台处理。先来看看看easyUI树的json字符串长什么样.

```json  
[{   
    "id":1,   
    "text":"Folder1",   
    "iconCls":"icon-save",   
    "children":[{   
        "text":"File1",   
        "checked":true  
    },{   
        "text":"Books",   
        "state":"open",   
        "attributes":{   
            "url":"/demo/book/abc",   
            "price":100   
        },   
        "children":[{   
            "text":"PhotoShop",   
            "checked":true  
        },{   
            "id": 8,   
            "text":"Sub Bookds",   
            "state":"closed"  
        }]   
    }]   
},{   
    "text":"Languages",   
    "state":"closed",   
    "children":[{   
        "text":"Java"  
    },{   
        "text":"C#"  
    }]   
}]
```

从中可以看出树控件每个节点都具备以下属性：

* id：节点ID，对加载远程数据很重要。
* text：显示节点文本。
* state：节点状态，'open' 或 'closed'，默认：'open'。如果为'closed'的时候，将不自动展开该节点。
* checked：表示该节点是否被选中。
* attributes: 被添加到节点的自定义属性。
* children: 一个节点数组声明了若干节点。

ok，明白了树节点的相关属性，我们便可以来构建树节点信息相关的实体类

**TreeNode.java**

```java
public class TreeNode{
    //显示节点的id  
    private Object id;  
    //节点状态
    private String state;
    //显示节点的名称
    private String text;
    //显示节点的图标
    private String icon;
    //额外的需要用名
    private String externName;
    //显示节点的父节点
    private Object parentId;
    //显示节点的子节点集合
    private List<TreeNode> children;
    //添加子节点的方法
    public void addChild(TreeNode node){
        if(this.children==null){
            children=new ArrayList<TreeNode>();
            children.add(node);
        }else{
            children.add(node);
        }
    }
    //set、get方法省略。。。
}
```

下面是跟我们的业务相关的实体类

**Bookorg.java**

```java
public class Bookorg  implements java.io.Serializable {    
     private BigDecimal bookorgid;
     private Orgcandidate orgcandidate;
     private Bookorg bookorg;
     private String orgcode;
     private String orgcnname;
     private String orgenname;
     private String orgcnshort;
     private String orgcnspell;
     private String orgenshort;
     private String orgenspell;
     private BigDecimal orglevel;
     private String orgcharger;
     private String orgemail;
     private String orgaddress;
     private String orgremark;
     private BigDecimal orgsortid;
     private String orgcancelflag;
     private Set bookorgs = new HashSet(0);
     private Set bookpersons = new HashSet(0);
     private Set addressbooks = new HashSet(0);
     //get、set方法省略。。。
}
```

## 前台页面

html代码

```html
<input id="parentEnterprise" name="parentEnterprise" style="width: 200px;height: 30px;"/>
```

js代码

```javascript
$("#parentEnterprise").combotree(
	{
		url : basePath + 'getDeptTree',
		panelWidth : 200,
		panelHeight : 250,
		lines:true,
		animate : false,
		multiple : false,
		required : true,
		missingMessage : '请选择上级机构!',
		onlyLeafCheck : true,
		cascadeCheck : false,
		onLoadSuccess : function(node, data) {
			if(node!=null){
				//设置父节点为展开状态
				$('#parentEnterprise').combotree('tree').tree('expand',node.target);
							}
						},
		onSelect : function(node) {
				$('#parentEnterprise').val(node.id);// 赋值
						},
		onBeforeExpand : function(node) {
				//将当前所点击的节点id传递到后台
				$('#parentEnterprise').combotree('tree').tree('options').url = basePath + 'getDeptTree?pid='+node.id;
						}
					});
```

## 后台代码

如果我们的所要构建的树的数据量不是很大，就可以进行同步加载，也就是一次性把树加载出来，但是如果数据量很大的话，一次性加载就会出现页面卡死的现象，这里就要采用异步加载的方式，也就是说，我们每次将我们要展开的节点的id传递到后台，查询他的下级机构，然后拼接成json字符串，传递到前台。

action层

```java
public String getTreeNode() {
		HttpServletRequest request = ServletActionContext.getRequest();
		//前台父节点id，为null加载根节点
		String pid = request.getParameter("pid");
		//获取当前机构下面的子机构的集合
		List<Bookorg> departmentList = departmentService.getTreeNode(pid);
		//定义构成tree json字符串
		String jsonStr = "";
		// 定义一个树形结构实体
		List<TreeNode> list = new ArrayList<TreeNode>();
		try {
			//循环遍历子机构
			for (Bookorg department : departmentList) {
				// 构建树节点信息
				TreeNode node = new TreeNode();
				//设置节点id
				node.setId(department.getBookorgid().longValue());
				//设置节点名称
				node.setText(department.getOrgcnname());
				//设置该节点父级节点id
				if (department.getBookorg() == null) {
					node.setParentId(null);
				} else {
					node.setParentId(department.getBookorg().getBookorgid()
							.longValue());
				}
				// 判断当前节点是否有子节点，node 的 state 为close 则节点设置为文件夹，为open 则为文件
				List<Bookorg> bookorgs = departmentService
						.getTreeNode(department.getBookorgid().toString());
				if (bookorgs != null && !bookorgs.isEmpty()) {
					node.setState("closed");
				} else {
					node.setState("open");
				}
				// 将node节点添加到集合中
				list.add(node);
			}

			//将集合转换成json字符串
			jsonStr = JsonUtil.listToJson(list);
			//返回到前台页面
			request.setAttribute("responseText", jsonStr);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return SUCCESS;
	}
```

service层

```java
@Override
	public List<Bookorg> getTreeNode(String pid) {
		StringBuffer hql=new StringBuffer();
		hql=hql.append("from Bookorg b where  ");
		if (pid == null || "".equals(pid)) {
			//返回跟节点
			hql=hql.append(" b.bookorg is null and b.orgcancelflag = 1 order by b.orgsortid asc ");
			return departmentDao.getTreeNode(hql,new Object[]{});
		}else {
			//异步加载当前id下的子节点
			hql=hql.append(" b.bookorg.bookorgid= ? and b.orgcancelflag = 1  order by b.orgsortid asc ");
			return departmentDao.getTreeNode(hql,new Object[]{new BigDecimal(pid)});
		}
	}
```

Dao层

```java
 public List getTreeNode(String hql, Object[] values) {
	    if (hql != null) {
	      return super.getHibernateTemplate().find(hql, values);
	    }
	    	return null;
	  }
```

## 最终效果

![](/img/20161021/2.png)

## 总结

到此为止树的构建基本上就结束了，其实树的构建本质上就是字符串的拼接，只要我们理解了树节点的相关属性，根据自己的业务需求就可以很快的构建一颗树出来，ok，今天就到此为止了。
