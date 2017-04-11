---
layout: post
title: "特殊情况下的java异常优化"
date: 2017-04-11 14:58:38 +0800
comments: true
categories: java
tags: [arthurhoo arthurhoo's blog java exception]
keywords: arthurhoo arthurhoo's blog java exception
---
>最近做了限流组件的开发，其中对限流场景下的异常抛出进行了性能优化，现记录下来以备自查，也和大家分享。

## 引言

在java中抛出异常使我们处理程序错误的一种手段，但是在某些特定的场景下，抛出异常也是我们业务逻辑的一部分。比如，在限流组件中，对超过阈值的请求抛出异常，通过快速失败来保证服务器的稳定也是一种手段。这个时候异常就成了限流组件业务逻辑的一部分。如何优化类似限流抛出异常的这种情况下的性能，是本文要讨论的重点。<!-- more -->


## 场景特点和Exception的性能问题

展开讨论之前我们先总结一下限流场景的异常抛出的特点，也规范一下我们讨论的范围：
1. 异常作为限流业务的一部分，并不关心堆栈，只关心异常类型和message
2. 限流情况下抛出异常频繁，会有大量的异常对象产生

我们都知道Java虚拟机在创建Exception对象的时候通常要比创建普通对象耗时，这主要是因为创建Exception及其子类对象的时候，jvm需要将异常堆栈捕获并填充到对象中。这个过程是相当耗时的。同时，在引言描述的场景中，限流情况下会抛出大量的异常，这些异常对象只使用一次，在抛出之后依然占用内存空间，这会增加Young GC的次数。

在我的场景中，限流是通过推送规则到服务器上执行。规则是应用维度隔离的，一个规则包含一类限流异常，一个应用通常十几条规则。有三个特点：

* 抛出的异常类型是我们预定的BizException类, 类型一样，不同的只是message，业务系统通过message来区分被限流的服务和场景。
* BizException的message是一条限流配置一个message
* 抛出的异常是业务需要的异常，对于业务系统来说有用的是异常的类型和message来判断是否限流和哪一个规则限流，对于异常堆栈并不关心

所以，我们的优化从两个方面着手：

* 减少创建Exception对象的时间
* 减少创建的Exception对象的数量


## 优化方案
### 减少异常创建的数量

我们只需要为每条限流规则创建一个不同message的异常就可以了。 这样异常的个数就固定了，大大节省了内存空间，减少GC次数。

`所以解决办法就是：将每一个配置的限流异常单例化。在限流服务启动的时候创建异常对象，存入一个静态map，形成一个异常池`

简化代码示例：
```
// 创建Exception的方法改为如下形式
public class BizExceptionFactory {

    private static final String DEFAULT_BIZ_EXCEPTION = "DEFAULT LIMIT EXCEPTION";

    private static final Map<String, BizException> exceptionMap = new ConcurrentHashMap<String, BizException>();

    public static BizException newInstance(String message){
        synchronized (BizExceptionFactory.class){
            BizException exception = exceptionMap.get(message);
            if(exception == null){
                exception = new BizException(message);
                exceptionMap.put(message,exception);
            }
            return exception;
        }
    }

}
```

限流后抛出异常的代码改为：
```
throw BizFactory.newInstance(message);
```

### 提升Exception对象创建性能

Exception对象创建的时候耗时要比普通对象更多，这个瓶颈就是，创建对象的时候需要获得线程栈的一个快照，然后通过fillInStackTrace填充堆栈到Exception对象。jdk中的代码如下：

```
public synchronized Throwable fillInStackTrace() {
        if (stackTrace != null ||
            backtrace != null /* Out of protocol state */ ) {
            fillInStackTrace(0);
            stackTrace = UNASSIGNED_STACK;
        }
        return this;
    }
```
去除填充堆栈的逻辑，就可以大大缩减创建Exception对象的时间。

在自定义的异常类中覆盖如下方法：

```
@Override
    public Throwable fillInStackTrace() {
        return this;
    }
```

浅薄认识，欢迎大家指正！
