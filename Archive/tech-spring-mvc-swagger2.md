---
title: Spring MVC中使用Swagger2构建Restful API
date: 2017-09-23 20:50:45
comments: true
tags:
- swagger2
categories:
- 编程
---

不知道写后台的同学有没有这样的烦劳，每次写完相关的接口，都要写相关的接口文档，然后跟前端小伙伴进行联调，过程很是繁琐和费时间。那为了解决这些问题，Swagger2 就是一个很好的解决方案，它与 spring mvc 整合后，我们只需要少量的注解，它便可以自动的帮我们生成一份 RESTful API 文档，大大的减轻了劳动力。

<!-- more -->

现在越来越多的项目开始进行前后端分离，那为了方便前后端进行通信，就需要一套 API 准则，RESTful API 是目前比较成熟的一套互联网应用程序的 API 设计理论，至于具体什么是 RESTful API，可以参考阮一峰老师的博文：[RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)，便会对RESTful API 有个大概的了解。下面整理下 Spring MVC 中整合 Swagger2，我这里的测试项目架构是 SSM ，如果是 Spring Boot 架构的项目，配置的时候注解稍微有点区别，在文章的代码中有注释。

# 添加 maven 依赖

```xml
<dependency>  
  <groupId>io.springfox</groupId>  
  <artifactId>springfox-swagger2</artifactId>  
  <version>2.4.0</version>  
</dependency>  
<dependency>  
  <groupId>io.springfox</groupId>  
  <artifactId>springfox-swagger-ui</artifactId>  
  <version>2.4.0</version>  
</dependency>
```

# 新增Swagger配置类

**SwaggerConfig.java**

```java
// @Configuration 这里需要注意，如果项目架构是SSM，那就不要加这个注解，如果是 spring boot 架构类型的项目，就必须加上这个注解，让 spring 加载该配置。
@EnableWebMvc // spring boot 项目不需要添加此注解，SSM 项目需要加上此注解，否则将会报错。
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket buildDocket(){
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(buildApiInfo())
                .select().apis(RequestHandlerSelectors.basePackage("com.leeyom.controller"))// controller路径。
                .paths(PathSelectors.any())
                .build();
    }

    // 配置 API 文档标题、描述、作者等等相关信息。
    private ApiInfo buildApiInfo(){
        return new ApiInfoBuilder()
                .title("XXX系统API接口文档")
                .termsOfServiceUrl("http://leeyom.top/")
                .description("Spring MVC中使用Swagger2构建Restful API")
                .contact(new Contact("leeyom", "http://leeyom.top/", "leeyomwang@gmail.com"))
                .build();

    }
}
```

> 再通过buildDocket函数创建Docket的Bean之后，buildApiInfo()用来创建该Api的基本信息（这些基本信息会展现在文档页面中）。select()函数返回一个ApiSelectorBuilder实例用来控制哪些接口暴露给Swagger来展现，本例采用指定扫描的包路径来定义，Swagger会扫描该包下所有Controller定义的API，并产生文档内容（除了被@ApiIgnore指定的请求）。 

# 配置spring-mvc.xml

装配 Swagger 配置文件的 bean，需要在 spring-mvc.xml 配置文件中添加如下一句：

```xml
<bean class="com.leeyom.common.util.SwaggerConfig"/>  
```

# 使用Swagger注解

```java
@RestController
@RequestMapping(value="/users")     
public class UserController {
    static Map<Long, User> users = Collections.synchronizedMap(new HashMap<Long, User>());
    @ApiOperation(value="获取用户列表", notes="")
    @RequestMapping(value={""}, method=RequestMethod.GET)
    public List<User> getUserList() {
        List<User> r = new ArrayList<User>(users.values());
        return r;
    }
    @ApiOperation(value="创建用户", notes="根据User对象创建用户")
    @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "User")
    @RequestMapping(value="", method=RequestMethod.POST)
    public String postUser(@RequestBody User user) {
        users.put(user.getId(), user);
        return "success";
    }
    @ApiOperation(value="获取用户详细信息", notes="根据url的id来获取用户详细信息")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long")
    @RequestMapping(value="/{id}", method=RequestMethod.GET)
    public User getUser(@PathVariable Long id) {
        return users.get(id);
    }
    @ApiOperation(value="更新用户详细信息", notes="根据url的id来指定更新对象，并根据传过来的user信息来更新用户详细信息")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long"),
            @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "User")
    })
    @RequestMapping(value="/{id}", method=RequestMethod.PUT)
    public String putUser(@PathVariable Long id, @RequestBody User user) {
        User u = users.get(id);
        u.setName(user.getName());
        u.setAge(user.getAge());
        users.put(id, u);
        return "success";
    }
    @ApiOperation(value="删除用户", notes="根据url的id来指定删除对象")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long")
    @RequestMapping(value="/{id}", method=RequestMethod.DELETE)
    public String deleteUser(@PathVariable Long id) {
        users.remove(id);
        return "success";
    }
}
```
在完成了上述配置后，其实已经可以生产文档内容，但是这样的文档主要针对请求本身，而描述主要来源于函数等命名产生，对用户并不友好，我们通常需要自己增加一些说明来丰富文档内容。如下所示，我们在 **controller** 中通过@ApiOperation注解来给API增加说明、通过@ApiImplicitParams、@ApiImplicitParam注解来给参数增加说明。

* **@ApiOperation**：给API增加说明。
* **@ApiImplicitParam**：给单个参数添加说明。
* **@ApiImplicitParams**：给多个参数添加说明。
* 更多swagger 注解参考[swagger常用注解说明](http://www.jianshu.com/p/12f4394462d5)。

# API文档访问
启动项目，然后访问[http://localhost:8080/你的项目名/swagger-ui.html](http://localhost:8080/swagger-ui.html)，出现如下界面，说明则整合成功。
![](http://s1.wailian.download/2017/10/31/restful-api.png)
那至此，spring mvc 整合 swagger 基本上就完毕了，有什么不懂的可以留言讨论~

# 参考

* [SpringMVC集成springfox-swagger2构建restful API](http://blog.csdn.net/u014231523/article/details/54411026)
* [swagger常用注解说明](http://www.jianshu.com/p/12f4394462d5)
* [Spring Boot中使用Swagger2构建强大的RESTful API文档](http://blog.didispace.com/springbootswagger2/)


