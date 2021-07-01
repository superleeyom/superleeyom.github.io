---
title: java 中 Integer 的缓存问题
date: 2017-09-09 16:59:39
comments: true
tags:  
- java
categories:
- 编程
---

曾在微信群里跟同学讨论一道关于 java 包装类 Integer 的面试题, 这个道题目虽然看似很简单, 但是如果没有仔细研读过 jdk 源码的人, 是很容易答错的, 那肯定自然我也是答错了的, 所以才有了今天这篇文章.

<!-- more --> 

首先来看下这个题目, 问输出的结果是多少?

```java
package test;    
    
public class Test {      
    public static void main(String[] args) {    
        Integer i1 = 127;    
        Integer i2 = 127;    
        System.out.println(i1 == i2);    
            
        Integer i3 = 128;    
        Integer i4 = 128;    
        System.out.println(i3 == i4);    
        
        Integer i5 = new Integer(100);
        Integer i6 = new Integer(100);
        System.out.println(i5 == i6);
        
        Integer i7 = 100;
        Integer i8 = new Integer(100);
        System.out.println(i7 == i8);
        
        int i9 = 100;
        Integer i10 = new Integer(100);
        System.out.println(i9 == i10);
    }    
}   
```
下面逐条分析结果

## i1 == i2 为 true
Integer i1 = 127 这种方式等价于`Integer i1 = Integer.valueOf(127).` 这里变涉及到Integer 缓存机制的问题, Integer 对于小数据(-128 ~ 127)是有缓存的, 在 jvm 初始化的时候, **-128 ~ 127 之间的数据就已经被缓存到本地的内存当中去了. 这样初始化-128 ~ 127之间的数字, 便会直接从内存中读取, 而不需要再创建对象. 所以 i1 和 i2 实际上引用的是同一个内存地址,** 那自然的结果也就是 true 了.

我们看 jdk 的源码, 查看`Integer.valueOf`方法:

```java
 public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```
可以看到，在执行valueOf的时候，会先去检查内存中是否存在该数字，如果存在的话，就直接从内存中取出返回，如果不存在的话，就新建一个Integer对象.

其中该缓存数据的初始化代码在:

```java
   private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

## i3 == i4 为 false
有了以上的解释我们已经知道, 128已经超乎了缓存数据范围, 所以会使用new新建个对象，那么i3和i4的内存地址就不一样了，结果就是false.

## i5 == i6 为 false
i5和i6的内存地址不一样，==的左右操作数如果是对象的话，那么比较的是引用的地址，new产生的对象一定是新的内存地址, 两者引用地址不一样,所以为 false.

## i7 == i8 为 false
i7是内存中的缓存数据, 有指定的内存地址, 而 i8是 new 出来的, 他指向的是新的内存地址, 二者之间指向的内存地址不一样, 所以结果为 false.

## i9 == i10 为 true
Integer 是 int 的包装类, Integer 类型的数据在跟 int 类型的数据做比较的时候, 会自动进行拆箱变成 int, 这样一来就是两个 int 类型的数据比较, 所以自然结果就是 true.

## 其他包装类有缓存机制吗?

答案肯定是! 以下做个总结:

| 包装类  | 缓存范围 |
| :-: | :-: |
| Boolean  | 全部缓存 |
| Byte | 全部缓存 |
| Character | <= 127缓存 |
| Short | -128 ~ 127缓存 |
|  Integer | -128 ~ 127缓存 |
| Long | -128 ~ 127缓存 |
| Float | 没有缓存 |
| Doulbe | 没有缓存 |



