---
title: SpringCloud Fegin超时重试源码
date: 2017.04.10 16:28:33
tags:
	- java
categories:
	- 程序人生
---
springCloud中最重要的就是微服务之间的调用，因为网络延迟或者调用超时会直接导致程序异常，因此超时的配置及处理就至关重要。

在开发过程中被调用的微服务打断点发现会又多次重试的情况，测试环境有的请求响应时间过长也会出现多次请求，网上查询了配置试了一下无果，决定自己看看源码。
本人使用的SpringCloud版本是Camden.SR3。

微服务间调用其实走的是http请求，debug了一下默认的ReadTimeout时间为5s，ConnectTimeout时间为2s，我使用的是Fegin进行微服务间调用，底层用的还是Ribbon，网上提到的配置如下

```
ribbon:
  ReadTimeout: 60000
  ConnectTimeout: 60000
  MaxAutoRetries: 0
  MaxAutoRetriesNextServer: 1
```
 超时时间是有效的但是重试的次数无效，如果直接使用ribbon应该是有效的。

下面开始测试：
在微服务B中建立测试方法，sleep 8s 确保请求超时

```java
@PostMapping("/testa")
    public Integer testee(){
        try {
            Thread.sleep(8000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 9;
    }
```
在微服务A中使用fegin调用此方法时看到有异常

看到在SynchronousMethodHandler中请求的方法

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = targetRequest(template);
    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }
    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      response.toBuilder().request(request).build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
     //出现异常后抛出RetryableException
      throw errorExecuting(request, e);
    }
```
出现异常后调用 throw errorExecuting(request, e) 抛出异常

在调用executeAndDecode的地方catch

```java
@Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
         //重试的地方
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```
retryer.continueOrPropagate(e); 这句就是关键继续跟进

```java
public void continueOrPropagate(RetryableException e) {
        //maxAttempts是构造方法传进来的大于重试次数抛出异常,否则继续循环执行请求
      if (attempt++ >= maxAttempts) {
        throw e;
      }
     ....
```
默认的Retryer构造器

```java
public Default() {
      this(100, SECONDS.toMillis(1), 5);
    }
```
第一个参数period是请求重试的间隔算法参数，第二个参数maxPeriod 是请求间隔最大时间，第三个参数是重试的次数。算法如下：

```java
 long nextMaxInterval() {
      long interval = (long) (period * Math.pow(1.5, attempt - 1));
      return interval > maxPeriod ? maxPeriod : interval;
    }
```

我们能否改写参数呢？我们再看看SpringCloud的文档中关于Retry的配置

![](http://upload-images.jianshu.io/upload_images/4702918-fe8f774565deec7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
新建一个配置类

```java
@Configuration
public class FeginConfig {
    @Bean
    public Retryer feginRetryer(){
        Retryer retryer = new Retryer.Default(100, SECONDS.toMillis(10), 3);
        return retryer;
    }
}
```
在feginClient是加入configuration的配置

```java
@FeignClient(value = "fund-server",fallback = FundClientHystrix.class,configuration = FeginConfig.class)
public interface FundClient 
```
重启重试，只调用了一次，Fegin重试次数解决。
我们再看看请求超时这里的参数

```java
@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);
                     //请求参数
			IClientConfig requestConfig = getClientConfig(options, clientName);
			return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
					requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
```
其中ReadTimeout 和 ConnectTimeout 读取的就是ribbon的配置，再来看一眼

```
ribbon:
  ReadTimeout: 60000
  ConnectTimeout: 60000
  MaxAutoRetries: 0
  MaxAutoRetriesNextServer: 1
```
![图片.png](http://upload-images.jianshu.io/upload_images/4702918-b565689507627852.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果想覆盖ribbon的超时设置可以在刚刚写的FeginConfig里注入下面的bean

```java
    @Bean
    public Request.Options feginOption(){
        Request.Options option = new Request.Options(7000,7000);
        return option;
    }
```

总结：使用开源的东西在弄不清问题出在哪时最好能看看源码，对原理的实现以及自己的编码思路都有很大的提升。

