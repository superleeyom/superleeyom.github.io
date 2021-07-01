---
title: Dubbo 异常的处理浅析
date: 2018-04-12 11:33:40
comments: true
tags:
- Dubbo
categories:
- 编程
---

在使用 Dubbo 进行服务治理的时候，如果消费者调用提供者的接口，接口内部出现异常，如果没有做处理的话，提供者会直接把异常抛给消费者，这样会出现的问题是，消费方无法反序列化相应异常，从而无法定位问题。而且关于这个问题，Dubbo 的官方文档也明确表示，应在服务实现中对消费方不关心的异常进行包装。

<!-- more -->

我们知道 Dubbo 的服务分为消费者和提供者，而消费者和提供者往往都不在一个系统中，所以消费者一般在调用服务端的接口时，通常会返回一个结果实体，来标明这一次请求操作是否成功。那这个实体的设计可以为：

```java
public class BaseResultDTO<T> {

    /**
     * 是否操作成功
     */
    private boolean success;

    /**
     * 提示信息
     */
    private String msg;
    /**
     * 操作结果
     */
    private T result;
}
```

那消费者调用提供者的接口的时候，就可以从返回的结果中得知此次调用是否成功，以及调用的结果是什么。但是这样会有一个问题，如果服务端抛异常了，这个时候就会将异常抛给消费者，那抛给消费者的时候，这里就出现给消费者一些迷惑性的异常。为什么这么说呢？

- 服务端与客户端，很可能不在同一个应用中，所以各自会依赖不同的 jar 包，比方说：服务端抛出了个 spring 的 `duplicateKeyException` ，但是客户端并没用引用 spring 的相关 jar 包，这样就会导致：抛出异常后，由于客户端没有依赖这个类，最终抛出个 `ClassNotDefError`，注意是 Error 不是 Exception。如果客户端只对 Exception 进行捕获的话，会导致直接抛到最顶层。可能日志、重试等都没了，这样无法准确通过日志去定位问题。

那最好的解决方案是，服务端对异常进行捕获，如果出现异常后，把异常信息转换成字符串，然后把异常信息返回到消费者，这个时候，这个实体可以设计为：

```java
public class BaseResultDTO<T> {
 
    /**
     * 是否操作成功
     */
    private boolean success;
 
    /**
     * 提示信息
     */
    private String msg;
    /**
     * 操作结果
     */
    private T result;
 
    /**
     * 异常堆栈信息
     */
    private String errorTrace;
}
```

提供者的接口方法内部可以这样写：

```java
public BaseResultDTO getUserById(Long userId) {
        BaseResultDTO<User> baseResultDTO = new BaseResultDTO<>();
        try {
            // do something
            User user = userService.getUserById(userId);
            baseResultDTO.setResult(user);
            baseResultDTO.setSuccess(true);
            baseResultDTO.setMsg("请求成功！");
        } catch (Exception e) {
            baseResultDTO.setErrorTrace(e.getMessage());
            baseResultDTO.setMsg("请求失败！");
            baseResultDTO.setSuccess(false);
        }
        return baseResultDTO;
    }
```

errorTrace 就是存储异常堆栈信息的属性，这样如果客户端检测到 success 为 false，这样就可以直接把 errorTrace 打到 log 中，方便定位问题。


