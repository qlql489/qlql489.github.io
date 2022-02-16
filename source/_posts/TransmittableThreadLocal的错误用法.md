---
title: TransmittableThreadLocal的错误用法
date: 2020-08-10 00:02:18
tags:
	- java
	- 线程
categories:
	- 程序人生
---

![pexels-this-is-zun-1194036](http://image.nianlun.tech/2022/02/16/54d1e01e7fdce6b99fe5291f657fe29e.jpg)

### 前言

ThreadLocal能够在单个线程中传递参数，使用可以用在系统参数的传递或者在链路跟踪中传递trace相关信息，需要说明的是单单使用ThreadLocal是不会出现ThreadLocal值线程共享的，但仅仅使用ThreadLocal还不够，如果代码中有使用异步，ThreadLocal就无能为力了，这时可以使用JDK自带的InheritableThreadLocal，这次ThreadLocal变量线程共享，就是因为使用了InheritableThreadLocal。

我们的项目使用springboot构建，想使用ThreadLocal来完成透传系统参数，这样所有接口和方法都不要显式的传递次参数，刚刚说到ThreadLocal无法解决异步传递问题，InheritableThreadLocal也只能解决新建线程的情况，无法在线程池的场景中使用，具体原因下次分析，经过调研我们使用阿里开源的TransmittableThreadLocal 
### 示例
我们先来看这个例子：

```java
public class TransmittableThreadLocalUtil {

    private static Logger logger = LoggerFactory.getLogger(TransmittableThreadLocalUtil.class);

    private static final TransmittableThreadLocal<Map<String, Object>> threadLocal = new TransmittableThreadLocal() {
        @Override
        protected Object initialValue() {
            return new HashMap(4);
        }
    };
    
    public static <T> T get(String key) {
        logger.info("get {}",key);
        Map map = (Map)threadLocal.get();
        return (T)map.get(key);
    }

    public static void set(String key, Object value) {
        Map map = (Map)threadLocal.get();
        map.put(key, value);
    }

    public static <T> T remove(String key) {
        Map map = (Map)threadLocal.get();
        return (T)map.remove(key);
    }
```

使用阿里的TransmittableThreadLocal封装了一个工具类，初始化一个map，有get、set、remove方法，其他方法省略。

```java
    @GetMapping("/testLocal2")
    public void test2(){
        logger.info("testLocal2 get "+ TransmittableThreadLocalUtil.get("test"));
    }

    @GetMapping("/testLocal3")
    public void test3(){
        TransmittableThreadLocalUtil.set("test","200");
    }
```

两个接口，一个设置test为200，另一个方法获取，springboot默认使用tomcat作为容器，我们知道tomcat在处理请求时使用了线程池，理论上2个接口的调用不会是同一个线程，我们看log输出可以确认：

```java
2020-08-09 22:59:36.765  INFO 86354 --- [nio-8080-exec-1] c.j.controller.ThreadLocalController     : testLocal3 test set 200
2020-08-09 22:59:37.755  INFO 86354 --- [nio-8080-exec-2] c.j.controller.ThreadLocalController     : testLocal2 get 200
```

先调用了/testLocal3接口线程名称是nio-8080-exec-2，再调用了/testLocal2，线程名称是nio-8080-exec-1，是两个线程

但是通过log可以看到第二个线程居然取到了第一个线程设置的ThreadLocal变量，这是怎么回事？ThreadLocal不是线程隔离的吗？

上面只是用一个小例子帮助大家理解，实际中我们使用拦截器在方法调用前获取系统参数放入ThreadLocal中，在调用结束时清理ThreadLocal变量，但因为同样有上面例子中的问题，我们的一此请求没结束时可能ThreadLocal中的值就被别的线程清理的，导致业务异常。

### 原因

这到底是什么原因导致的呢，在翻看了TransmittableThreadLocal的源码以及项目中的使用方式后找到了原因。

TransmittableThreadLocal继承自jdk的**InheritableThreadLocal**，而**InheritableThreadLocal**的原理是父线程创建子线程时会将父线程的ThreadLocalMap复制到子线程中，这个复制是引用复制，也就是说子线程可以修改父线程ThreadLocal中的变量，但这也解释不了为什么ThreadLoal会线程共享，tomcat线程池中的线程都是子线程啊，那只可能是父线程出现了问题，再看一下项目代码，找到了原因。

因为在service的静态变量中使用了TransmittableThreadLocalUtil的get方法初始化了那个map，而springboot在加载service时使用的是main主线程，是所有线程的父线程，导致所有的子线程通过TransmittableThreadLocal获取的Map是同一个引用，当然是线程共享了！

例子中的service示例：

```java
@Service
public class TestService {
	//静态变量的调用
    private String value = TransmittableThreadLocalUtil.get("test");

    public void testMethod(){

    }

}
```
get方法中有log，再看一眼启动log

```java
2020-08-09 22:59:24.104  INFO 86354 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 2164 ms
2020-08-09 22:59:24.202  INFO 86354 --- [           main] c.j.util.TransmittableThreadLocalUtil    : get test
2020-08-09 22:59:24.808  INFO 86354 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 
```

日志第二行，main线程调用了TransmittableThreadLocalUtil，确定了推断。

其中一种解决方式是在TransmittableThreadLocalUtil的初始化TransmittableThreadLocal对象时复写childValue方法

```java
private static final TransmittableThreadLocal<Map<String, Object>> threadLocal = new TransmittableThreadLocal() {
        @Override
        protected Object initialValue() {
            return new HashMap(4);
        }
  			@Override
        protected Object childValue(Object parentValue) {
            if(parentValue instanceof Map){
                return new HashMap<>((Map)parentValue);
            }
            return parentValue;
        }
    };
```
### 总结
1、ThreadLocal不会出现线程间共享的情况。 

2、InheritableThreadLocal的原理是新建子线程时将父线程ThreadLocal复制到子线程中，是引用复制，会导致父子线程都能操作那个引用。

3、问题不是TransmittableThreadLocal引起的，是因为错误使用了InheritableThreadLocal，TransmittableThreadLocal只是继承自InheritableThreadLocal。

为什么复写`childValue`方法就可以解决呢，ThreadLocal有没有别的坑，InheritableThreadLocal以及ThreadLocal中相关的源码是如何处理的呢，期待下一篇吧~