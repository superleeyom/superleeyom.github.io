---
title: swagger注解总结
date: 2017-09-25 22:35:59
comments: true
tags:
- swagger注解
categories:
- 编程
---

随着互联网技术的发展，现在的网站架构基本都由原来的后端渲染，变成了前端渲染、先后端分离的形态，而且前端技术和后端技术在各自的道路上越走越远。前端和后端的唯一联系，变成了API接口，API文档变成了前后端开发人员联系的纽带，变得越来越重要，swagger就是一款让你更好的书写API文档的框架，本篇主要总结常用的swagger注解。

<!-- more -->

# 快速预览

| 名称 | 描述 |
| :-: | :-: |
| [@Api](#Api) | 将当前注解所在的类作为 Swagger 资源 |
| [@ApiImplicitParam](#ApiImplicitParam) | 代表某个接口中的一个参数 |
| [@ApiImplicitParams](#ApiImplicitParams) | 代表某个接口中的多个参数，由多个ApiImplicitParam对象组成 |
| [@ApiModel](#ApiModel) | 为 Swagger 模型提供额外的信息 |
| [@ApiModelProperty](#ApiModelProperty) | 添加和使用模型属性数据 |
| [@ApiOperation](#ApiOperation) | 对特定操作路径的 http 方法进行描述 |
| [@ApiParam](#ApiParam) | 为操作参数添加其他元数据 |
| [@ApiResponse](#ApiResponse) | 描述一个操作可能的回应 |
| [@ApiResponses](#ApiResponses) | 描述一个操作的多个回应，由多个ApiResponse对象组成 |
| [@Authorization](#Authorization) | 声明要在资源或操作上使用的授权方案。 |
| [@AuthorizationScope](#AuthorizationScope) | 描述一个OAuth2的授权范围 |
| [@ResponseHeader](#ResponseHeader) | 表示可以作为响应的部分表头 |

最新版本还增加了一些用于 Swagger 定义级别添加扩展和元数据的注释：


| 名称 | 描述 |
| :-: | :-: |
| [@SwaggerDefinition](#SwaggerDefinition) | 定义级别属性将添加到生成的 Swagger定义中 |
| [@Info](#Info) | 一个 Swagger 定义通用的元数据 |
| [@Contact](#Contact) | 为一个 Swagger 定义添加联系人相关属性 |
| [@License](#License) | 为一个Swagger 定义添加许可证 |
| [@Extension](#Extension) | 添加带有包含属性的扩展 |
| [@ExtensionProperty](#ExtensionProperty) | 向扩展添加自定义属性 |

# 注解详解

## <a id="Api">@Api</a>
用来标记当前Controller类为swagger文档资源，使用如下：

```java
@Api(value = "/CounterfeitSellerPurchaseAccount", description = "跟卖订单购买账号接口")
public class CounterfeitSellerPurchaseAccountHandler {
}
```
属性配置：

| 属性名称 | 备注 |
| :-: | :-: |
| value | url的路径值 |
| tags | 如果设置这个值、value的值会被覆盖 |
| description | 对api资源的描述 |
| basePath | 基本路径可以不配置 |
| position | 如果配置多个Api 想改变显示的顺序位置 |
| produces | For example, "application/json, application/xml" |
| consumes | For example, "application/json, application/xml" |
| protocols | Possible values: http, https, ws, wss. |
| authorizations | 高级特性认证时配置 |
| hidden | 配置为true 将在文档中隐藏 |

## <a id="ApiOperation">@ApiOperation</a>
该注解主要作用在方法上，对该方法进行描述，说明方法的作用，使用如下：

```java
@ApiOperation(value = "新增购买账号", notes = "根据purchaseAccount创建新的购买账户")
@RequestMapping(value = "/addPurchaseAccount", method = RequestMethod.PUT)
public String addPurchaseAccount(@ApiParam(value = "新增购买账号实体", required = true) @RequestBody CounterfeitSellerPurchaseAccount purchaseAccount, HttpServletRequest request) {
}
```
属性配置：

| 属性名称 | 备注 |
| :-: | :-: |
| value	| url的路径值|
| tags| 	如果设置这个值、value的值会被覆盖|
| description| 	对api资源的描述|
| basePath| 	基本路径可以不配置|
| position	| 如果配置多个Api 想改变显示的顺序位置|
| produces| 	For example, "application/json, application/xml"|
| consumes	| For example, "application/json, application/xml"|
| protocols	| Possible values: http, https, ws, wss.|
| authorizations| 	高级特性认证时配置|
| hidden	| 配置为true 将在文档中隐藏|
| response| 	返回的对象|
| responseContainer| 	这些对象是有效的 "List", "Set" or "Map".，其他无效|
| httpMethod| 	"GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH"|
| code| 	http的状态码 默认 200|
| extensions| 	扩展属性|

## <a id="ApiParam">@ApiParam</a>

该标签的主要作用是描述请求参数的属性，感觉它的作用跟[@ApiImplicitParam](#ApiImplicitParam)有点类似，使用示例如下：

```java
@ApiOperation(value = "新增购买账号", notes = "根据purchaseAccount创建新的购买账户")
    @RequestMapping(value = "/addPurchaseAccount", method = RequestMethod.PUT)
    public String addPurchaseAccount(@ApiParam(value = "新增购买账号实体", required = true) @RequestBody CounterfeitSellerPurchaseAccount purchaseAccount, HttpServletRequest request) {
}
```
属性配置：

| 属性名称| 	备注|
| :-: | :-: |
| name| 	属性名称|
| value| 	属性值|
| defaultValue| 	默认属性值|
| allowableValues| 	可以不配置|
| required| 	是否属性必填|
| access| 	不过多描述|
| allowMultiple| 	默认为false|
| hidden	| 隐藏该属性|
| example| 	举例子|

## <a id="ApiResponse">@ApiResponse</a>

响应设置，比如当前请求响应为400，则提示”Invalid Order”，使用示例：

```java
@RequestMapping(value = "/order", method = POST)
@ApiOperation(value = "Place an order for a pet", response = Order.class)
@ApiResponse(code = 400, message = "Invalid Order")
public ResponseEntity<String> placeOrder(
      @ApiParam(value = "order placed for purchasing the pet", required = true) Order order) {
    storeData.add(order);
    return ok("");
  }
```

属性说明：

| 属性名称| 	备注|
| :-: | :-: |
| code| 	http的状态码|
| message| 	描述|
| response| 	默认响应类 Void|
| reference| 	参考ApiOperation中配置|
| responseHeaders	| 参考 ResponseHeader 属性配置说明|
| responseContainer| 	参考ApiOperation中配置|

## <a id="ApiResponses">@ApiResponses</a>

设置一组响应集，使用示例：

```java
@RequestMapping(value = "/order", method = POST)
@ApiOperation(value = "Place an order for a pet", response = Order.class)
@ApiResponses({
    @ApiResponse(code = 400, message = "Invalid Order"),
    @ApiResponse(code = 500, message = "Server Error")
})
public ResponseEntity<String> placeOrder(
      @ApiParam(value = "order placed for purchasing the pet", required = true) Order order) {
    storeData.add(order);
    return ok("");
  }
```

## <a id="ApiImplicitParam">@ApiImplicitParam</a>

对方法的请求参数的描述，使用示例如下：

```java
@ApiOperation(value = "获取购买账户详情", notes = "根据url的accountId来获取购买账户详情信息")
    @ApiImplicitParam(name = "accountId", value = "账户ID", required = true, dataType = "long", paramType = "path")
    @RequestMapping(value = "/getPurchaseAccount/{accountId}", method = RequestMethod.GET)
    @ResponseBody
    public CounterfeitSellerPurchaseAccount getPurchaseAccount(@PathVariable long accountId) {
        return sellerPurchaseAccountService.getPurchaseAccountById(accountId);
    }
```

属性说明：


* paramType：参数放在哪个地方
    * header：请求参数的获取：@RequestHeader
    * query：请求参数的获取：@RequestParam
    * path：（用于restful接口）url后面跟着的请求参数的获取：@PathVariable
    * body：（不常用）
    * form：（不常用）
* name：参数名
* dataType：参数类型，long，string，int
* required：参数是否必须传
* value：参数的意思
* defaultValue：参数的默认值

## <a id="ApiImplicitParams">@ApiImplicitParams</a>

这个注解自然就是说一组参数的使用说明，使用示例如下：

```java
@ApiOperation(value = "获取购买账号列表", notes = "获取购买账号列表")
    @ApiImplicitParams({
            @ApiImplicitParam(paramType = "query", name = "limit", value = "分页参数", required = true, dataType = "int"),
            @ApiImplicitParam(paramType = "query", name = "offset", value = "分页参数2", required = true, dataType = "int"),
            @ApiImplicitParam(paramType = "query", name = "countryId", value = "国家id，查询参数，默认为null", required = false, dataType = "int"),
            @ApiImplicitParam(paramType = "query", name = "deliveryFullName", value = "账号名字，查询参数，默认为null", required = false, dataType = "string")
    })
    @RequestMapping(value = "/getPurchaseAccountList", method = RequestMethod.GET)
    public String getPurchaseAccountList(@RequestParam(value = "countryId", required = false) Integer countryId, @RequestParam(value = "deliveryFullName", required = false) String deliveryFullName, @RequestParam(value = "limit") Integer limit, @RequestParam(value = "offset") Integer offset, HttpServletRequest request) {
}
```

## <a id="ResponseHeader">@ResponseHeader</a>
用于描述一个响应的请求头，使用示例如下：

```java
@ApiResponses(value = {
      @ApiResponse(code = 400, message = "Invalid ID supplied",
                   responseHeaders = @ResponseHeader(name = "X-Rack-Cache", description = "Explains whether or not a cache was used", response = Boolean.class)),
      @ApiResponse(code = 404, message = "Pet not found") })
  public Response getPetById(...) {...}
```

## <a id="ApiModel">@ApiModel</a>

如果我么的请求参数是一个比较复杂的对象，比如 User 对象，就需要我们使用该属性对 User 对象进行描述，使用示例如下：

```java
@ApiModel(value = "User", description = "用户对象")
public class User {
    @ApiModelProperty(value = "ID")
    private Integer id;
    @ApiModelProperty(value = "姓名")
    private String name;
    @ApiModelProperty(value = "地址")
    private String address;
    @ApiModelProperty(value = "年龄",access = "hidden")
    private int age;
    @ApiModelProperty(value = "性别")
    private int sex;
    .......
}
```

```java
@ApiOperation(value="创建用户-传递复杂对象", notes="传递复杂对象DTO，json格式传递数据",produces = "application/json")
@RequestMapping(value="/users-3", method= RequestMethod.POST)
//json格式传递对象使用RequestBody注解
public User postUser3(@RequestBody User user) {
    users.put(user.getId(),user);
    return user;
}
```

## <a id="ApiModelProperty">@ApiModelProperty</a>
该注解用于描述复杂请求对象的属性，参考[@ApiModel](#ApiModel)
