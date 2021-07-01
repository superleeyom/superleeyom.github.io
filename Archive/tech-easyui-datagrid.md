---
title: easyUI datagrid 动态表头的实现
date: 2016-11-26 19:46:09
comments: true
categories:
- 编程
tags:
- easyUI
---

最近在项目中，需要用easyUI 的datagrid组件去实现一个表报的功能。我们以前用datagrid都是前台约定好字段，然后我们返回的json字符串与前台字段相对应便可以生成数据表格，但是这次却不同，datagrid的表头是动态的，为了实现这个功能，琢磨了些时间，然后在此总结一下。

<!--more-->

## 场景
点击左侧的配置项名称，右侧生成统计报表，报表可以是一维的，也可能是二维的，表头动态变化的，如下面所示：

![2016112619244QQ20161126-0@2x.png](http://image.leeyom.top/2016112619244QQ20161126-0@2x.png)

## 思路

从后台读取列名称，在`$("#dg2").datagrid({})`时，并不指定url属性，不向服务器端发送请求，在datagrid之后，通过ajax向服务器发送请求，并接收绑定列名称，和内容的json。

## 实现

### 前台

1. jsp页面

  ```html
<div title="表格统计结果">
	<table id="dg2"></table>
</div>
```

2. JavaScript

  ```js
function queryTable(configId,configName){
	$("#tableHead").html(configName);
	//加载表格数据和动态表头
	$.ajax({
        url:basePath + 'statisticAnalysis/loadTableData.action',
        data : {
			'configId' : configId
		},
		cache : false,
		dataType : "text",
        success:function(msg){
        	var result = eval('(' + msg + ')');
        	//统计表格
        	$('#dg2').datagrid({
        		fit : true,
        		fitColumns : true,
        		striped : true,
        		singleSelect: true,
            	//填充动态表头
        		columns:result[0].columns,
        		pagination : false,
        		loadMsg : "正在努力加载中......",
        	});
          	//填充表格数据
        	$('#dg2').datagrid("loadData",result[1]);
        }
  });
}
```

### 后台

考虑到我的表报的复杂性，因为有两个维度，其实就是说，他的行(x)和列(z)是变化的，所以，我在通过webservice调取接口获取的数据之前，跟写接口的同事约定好我要的数据的json格式，如下：

```json
{
        "x":["2016-01","2016-02","2016-03","2016-04"],
        "z":["重大级别","较大级别","一般级别","未达级别"],
        "重大级别_2016-01":"34",
        "较大级别_2016-01":"56",
        "一般级别_2016-01":"98",
        "未达级别_2016-01":"23",
        "重大级别_2016-02":"67",
        "较大级别_2016-02":"112",
        "一般级别_2016-02":"234",
        "未达级别_2016-02":"25",
        "重大级别_2016-03":"67",
        "较大级别_2016-03":"112",
        "一般级别_2016-03":"234",
        "未达级别_2016-03":"25",
        "重大级别_2016-04":"76",
        "较大级别_2016-04":"116",
        "一般级别_2016-04":"115",
        "未达级别_2016-04":"25"
}
```

* x ：就是动态的表头数据
* z ：可以理解为列表头
* z以下： 这些就是表格数据，他们的键是根据x和z两两组合而成，值就是统计值，比如"重大级别_2016-01"代表的意思就是2016年1月份重大级别的事件发生的数量。

代码如下：

```java
public String loadTableData() {
		HttpServletRequest request = ServletActionContext.getRequest();
		//返回的json字符串，这里我就先暂时用假数据
		String jsonStr2 = "{"
				+ "\"x\":[\"2016-01\",\"2016-02\",\"2016-03\",\"2016-04\"],"
				+ "\"z\":[\"重大级别\",\"较大级别\",\"一般级别\",\"未达级别\"],"
				+ "\"重大级别_2016-01\":\"34\"," + "\"较大级别_2016-01\":\"56\","
				+ "\"一般级别_2016-01\":\"98\"," + "\"未达级别_2016-01\":\"23\","
				+ "\"重大级别_2016-02\":\"67\"," + "\"较大级别_2016-02\":\"112\","
				+ "\"一般级别_2016-02\":\"234\"," + "\"未达级别_2016-02\":\"25\","
				+ "\"重大级别_2016-03\":\"67\"," + "\"较大级别_2016-03\":\"112\","
				+ "\"一般级别_2016-03\":\"234\"," + "\"未达级别_2016-03\":\"25\","
				+ "\"重大级别_2016-04\":\"76\"," + "\"较大级别_2016-04\":\"116\","
				+ "\"一般级别_2016-04\":\"115\"," + "\"未达级别_2016-04\":\"25\"" + "}";
		//解析json字符串
		JSONObject jsonobj = JSONObject.fromObject(jsonStr2);
		JSONArray xAxle = jsonobj.getJSONArray("x");
		JSONArray zAxle = jsonobj.getJSONArray("z");

		// 动态设置表头
		String columns = "{\"columns\":[[";
		columns += ("{\"field\":\"zAxle\",\"title\":\"\",\"align\":\"center\",\"width\":50},");
		for (int i = 0; i < xAxle.size(); i++) {
			String xString = xAxle.getString(i);
			columns += ("{\"field\":\"x_" + i + "\",\"title\":\"" + xString + "\",\"align\":\"center\",\"width\":50},");
		}
		if (columns.endsWith(",")) {
			columns = columns.substring(0, columns.length() - 1);
			columns += "]]}";
		}

		// 动态设置表数据(需要考虑z轴为null的情况)
		String rows = "";
		if (zAxle.size() != 0) {
			rows = "{\"total\":" + zAxle.size() + ",\"rows\":[";
			for (int i = 0; i < zAxle.size(); i++) {
				String subRows = "{\"zAxle\":\"" + zAxle.getString(i) + "\",";
				for (int j = 0; j < xAxle.size(); j++) {
					subRows += "\"x_"
							+ j
							+ "\":\""
							+ jsonobj.get(zAxle.getString(i) + "_"
									+ xAxle.getString(j)) + "\",";
				}

				if (subRows.endsWith(",")) {
					subRows = subRows.substring(0, subRows.length() - 1);
					subRows += "},";
				}
				rows += subRows;
			}

			if (rows.endsWith(",")) {
				rows = rows.substring(0, rows.length() - 1);
				rows += "]}";
			}
		} else {
			rows = "{\"total\":" + 1 + ",\"rows\":[";
			String subRows = "{\"zAxle\":\"" + "数量" + "\",";
			for (int j = 0; j < xAxle.size(); j++) {
				subRows += "\"x_" + j + "\":\""
						+ jsonobj.get(xAxle.getString(j)) + "\",";
			}

			if (subRows.endsWith(",")) {
				subRows = subRows.substring(0, subRows.length() - 1);
				subRows += "},";
			}
			rows += subRows;

			if (rows.endsWith(",")) {
				rows = rows.substring(0, rows.length() - 1);
				rows += "]}";
			}
		}

		String mesg = "[" + columns + "," + rows + "]";
		System.out.println(mesg);
		request.setAttribute("responseText", mesg);
		return SUCCESS;
	}
```

最后返回前台的json数据如下：

```json
[
    {
        "columns":[
            [
                {
                    "field":"zAxle",
                    "title":"",
                    "align":"center",
                    "width":50
                },
                {
                    "field":"x_0",
                    "title":"2016-01",
                    "align":"center",
                    "width":50
                },
                {
                    "field":"x_1",
                    "title":"2016-02",
                    "align":"center",
                    "width":50
                },
                {
                    "field":"x_2",
                    "title":"2016-03",
                    "align":"center",
                    "width":50
                },
                {
                    "field":"x_3",
                    "title":"2016-04",
                    "align":"center",
                    "width":50
                }
            ]
        ]
    },
    {
        "total":4,
        "rows":[
            {
                "zAxle":"重大级别",
                "x_0":"34",
                "x_1":"67",
                "x_2":"67",
                "x_3":"76"
            },
            {
                "zAxle":"较大级别",
                "x_0":"56",
                "x_1":"112",
                "x_2":"112",
                "x_3":"116"
            },
            {
                "zAxle":"一般级别",
                "x_0":"98",
                "x_1":"234",
                "x_2":"234",
                "x_3":"115"
            },
            {
                "zAxle":"未达级别",
                "x_0":"23",
                "x_1":"25",
                "x_2":"25",
                "x_3":"25"
            }
        ]
    }
]
```

## 总结

其实思路很简单，关键在于我们要约定好我们的数据格式，然后将我们拿到的数据，再进一步的封装成前台datagrid需要的数据格式。当初一直卡在数据该如何去约定，我之所以这么约定数据格式，是因为不仅datagrid是动态的，同时在用echarts渲染成柱状图的时候，也是动态的，他也要需要特定的数据格式。为了方便后期遇到同样的问题，特此记录一下。
