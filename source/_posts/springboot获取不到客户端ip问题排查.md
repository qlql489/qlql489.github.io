---
title: springboot获取不到客户端ip问题排查
date: 2022-11-19 15:38:36
tags:
	- java
	- springboot
	- 问题排查
categories:
	- 程序人生
---

### 一、现象

springboot从2.0.2升级到 2.5.7后线上环境无法通过request.getHeader("x-forwarded-for")获取客户端ip地址，测试环境正常，开发环境也异常

### 二、结论

springboot 2.5.7版本中CloudPlatform多了Kubernetes platform的类型识别，如果使用的是内嵌的tomcat，在k8s环境中会自动添加了tomcat的RemoteIpValve，线上环境的httpHeader(x-forwarded-for)只有一个，没有代理ip信息，按RemoteIpValve的逻辑，x-forwarded-for头信息会被删除。

### 三、排查流程

#### 1、抓包看请求与现象

dev环境抓包

![image-20221119153546617](http://image.nianlun.tech/2022/11/19/a30167670facd04fb59cd2cc434e458b.png)

可以看到抓包中有x-forwarded-for，但是一个内网ip

#### 2、本地调试

使用develop分支本地模拟dev环境请求，debug发现可以获取到ip地址

#### 3、搜索springboot丢失x-forwarded-for的原因

搜索到文章https://loveyu.org/5951.html

看到了删除header的调用 **removeHeader**方法，但没有写是哪个类，搜索代码发现是内嵌tomcat的RemoteIpValve类中调用，大致看了下逻辑其中有删除header x-forwarded-for的代码，但打断点debug发现不会走到RemoteIpValve中

#### 4、查看RemoteIpValve的执行逻辑

查看上面文章提到的配置：server.forward-headers-strategy

通过搜索发现，是在spring的配置类org.springframework.boot.autoconfigure.web.ServerProperties中

```
/**
 * Strategy for handling X-Forwarded-* headers.
 */
private ForwardHeadersStrategy forwardHeadersStrategy;
```

get方法会在TomcatWebServerFactoryCustomizer类的getOrDeduceUseForwardHeaders方法中调用

```
private void customizeRemoteIpValve(ConfigurableTomcatWebServerFactory factory) {
   Remoteip remoteIpProperties = this.serverProperties.getTomcat().getRemoteip();
   String protocolHeader = remoteIpProperties.getProtocolHeader();
   String remoteIpHeader = remoteIpProperties.getRemoteIpHeader();
   // For back compatibility the valve is also enabled if protocol-header is set
   if (StringUtils.hasText(protocolHeader) || StringUtils.hasText(remoteIpHeader)
         || getOrDeduceUseForwardHeaders()) {
      RemoteIpValve valve = new RemoteIpValve();
      valve.setProtocolHeader(StringUtils.hasLength(protocolHeader) ? protocolHeader : "X-Forwarded-Proto");
      if (StringUtils.hasLength(remoteIpHeader)) {
         valve.setRemoteIpHeader(remoteIpHeader);
      }
      // The internal proxies default to a list of "safe" internal IP addresses
      valve.setInternalProxies(remoteIpProperties.getInternalProxies());
      try {
         valve.setHostHeader(remoteIpProperties.getHostHeader());
      }
      catch (NoSuchMethodError ex) {
         // Avoid failure with war deployments to Tomcat 8.5 before 8.5.44 and
         // Tomcat 9 before 9.0.23
      }
      valve.setPortHeader(remoteIpProperties.getPortHeader());
      valve.setProtocolHeaderHttpsValue(remoteIpProperties.getProtocolHeaderHttpsValue());
      // ... so it's safe to add this valve by default.
      factory.addEngineValves(valve);
   }
}
 
private boolean getOrDeduceUseForwardHeaders() {
   if (this.serverProperties.getForwardHeadersStrategy() == null) {
      CloudPlatform platform = CloudPlatform.getActive(this.environment);
      return platform != null && platform.isUsingForwardHeaders();
   }
   return this.serverProperties.getForwardHeadersStrategy().equals(ServerProperties.ForwardHeadersStrategy.NATIVE);
}
```

getOrDeduceUseForwardHeaders方法逻辑

1、如果没有配置forwardHeadersStrategy则判断目前的环境

org.springframework.boot.cloud.CloudPlatform#getActive

```
public static CloudPlatform getActive(Environment environment) {
   if (environment != null) {
      for (CloudPlatform cloudPlatform : values()) {
         if (cloudPlatform.isActive(environment)) {
            return cloudPlatform;
         }
      }
   }
   return null;
}
```

CloudPlatform枚举类中可以看到比之前的springboot版本多了KUBERNETES的枚举，也就是在k8s环境CloudPlatform.getActive(this.environment)返回的不为空，isUsingForwardHeaders返回也为true

```
public boolean isUsingForwardHeaders() {
   return true;
}
```

2、如果配置了则判断是否为NATIVE

新版本通过k8s环境判断getOrDeduceUseForwardHeaders方法返回true

getOrDeduceUseForwardHeaders返回为true，在customizeRemoteIpValve方法中就会添加RemoteIpValve

#### 5、为什么测试环境没事，dev和线上都有问题

通过arthus查看dev和测试环境的调用栈，发现调用栈不同

dev调用栈

```
`---ts=2022-11-14 20:45:50;thread_name=http-nio-8080-exec-1;id=5b;is_daemon=true;priority=5;TCCL=org.springframework.boot.loader.LaunchedURLClassLoader@3d24753a
    `---[1.786075ms] org.apache.catalina.valves.RemoteIpValve:invoke()
        +---[0.54% 0.009643ms ] org.apache.catalina.connector.Request:getRemoteAddr() #613
        +---[0.17% 0.002994ms ] org.apache.catalina.connector.Request:getRemoteHost() #614
        +---[0.19% 0.003369ms ] org.apache.catalina.connector.Request:getScheme() #615
        +---[0.15% 0.002714ms ] org.apache.catalina.connector.Request:isSecure() #616
        +---[0.17% 0.003068ms ] org.apache.catalina.connector.Request:getServerName() #617
        +---[0.15% 0.002709ms ] org.apache.catalina.valves.RemoteIpValve:isChangeLocalName() #618
        +---[0.16% 0.002895ms ] org.apache.catalina.connector.Request:getServerPort() #619
        +---[0.22% 0.003994ms ] org.apache.catalina.connector.Request:getLocalPort() #620
        +---[0.18% 0.00325ms ] org.apache.catalina.connector.Request:getHeader() #621
        +---[0.16% 0.002793ms ] org.apache.catalina.connector.Request:getHeader() #622
        +---[0.21% 0.00372ms ] org.apache.catalina.connector.Request:getHeaders() #632
        +---[0.18% 0.003182ms ] org.apache.catalina.valves.RemoteIpValve:commaDelimitedListToStringArray() #640
        +---[0.14% 0.002488ms ] org.apache.catalina.connector.Request:getHeader() #700
        +---[0.13% 0.002356ms ] org.apache.catalina.connector.Request:getHeader() #716
        +---[0.28% 0.004932ms ] org.apache.catalina.connector.Request:setAttribute() #736
        +---[0.14% 0.002555ms ] org.apache.juli.logging.Log:isDebugEnabled() #738
        +---[0.10% 0.001796ms ] org.apache.catalina.connector.Request:getRemoteAddr() #756
        +---[0.15% 0.002711ms ] org.apache.catalina.connector.Request:setAttribute() #755
        +---[0.13% 0.002321ms ] org.apache.catalina.connector.Request:getRemoteAddr() #758
        +---[0.12% 0.002186ms ] org.apache.catalina.connector.Request:setAttribute() #757
        +---[0.11% 0.001972ms ] org.apache.catalina.connector.Request:getRemoteHost() #760
        +---[0.12% 0.002058ms ] org.apache.catalina.connector.Request:setAttribute() #759
        +---[0.19% 0.003334ms ] org.apache.catalina.connector.Request:getProtocol() #762
        +---[0.12% 0.00218ms ] org.apache.catalina.connector.Request:setAttribute() #761
        +---[0.12% 0.002192ms ] org.apache.catalina.connector.Request:getServerName() #764
        +---[0.12% 0.00212ms ] org.apache.catalina.connector.Request:setAttribute() #763
        +---[0.11% 0.002021ms ] org.apache.catalina.connector.Request:getServerPort() #766
        +---[0.14% 0.002423ms ] org.apache.catalina.connector.Request:setAttribute() #765
        +---[0.39% 0.007047ms ] org.apache.catalina.valves.RemoteIpValve:getNext() #769
        +---[79.19% 1.414379ms ] org.apache.catalina.Valve:invoke() #769
        +---[0.23% 0.00409ms ] org.apache.catalina.connector.Request:setRemoteAddr() #771
        +---[0.16% 0.002866ms ] org.apache.catalina.connector.Request:setRemoteHost() #772
        +---[0.15% 0.002725ms ] org.apache.catalina.connector.Request:setSecure() #773
        +---[0.20% 0.00363ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #774
        +---[0.19% 0.003448ms ] org.apache.coyote.Request:scheme() #774
        +---[0.14% 0.002545ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #774
        +---[0.16% 0.002889ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #775
        +---[0.19% 0.00342ms ] org.apache.coyote.Request:serverName() #775
        +---[0.17% 0.00297ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #775
        +---[0.22% 0.003904ms ] org.apache.catalina.valves.RemoteIpValve:isChangeLocalName() #776
        +---[0.20% 0.00363ms ] org.apache.catalina.connector.Request:setServerPort() #779
        +---[0.17% 0.003065ms ] org.apache.catalina.connector.Request:setLocalPort() #780
        +---[0.16% 0.002877ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #782
        +---[0.18% 0.003204ms ] org.apache.coyote.Request:getMimeHeaders() #782
        +---[0.22% 0.003937ms ] org.apache.tomcat.util.http.MimeHeaders:removeHeader() #784
        `---[0.17% 0.003077ms ] org.apache.tomcat.util.http.MimeHeaders:removeHeader() #790
```

测试环境调用栈

```
`---ts=2022-11-14 21:02:40;thread_name=http-nio-8080-exec-5;id=aa;is_daemon=true;priority=5;TCCL=org.springframework.boot.loader.LaunchedURLClassLoader@42f85fa4
    `---[100.85578ms] org.apache.catalina.valves.RemoteIpValve:invoke()
        +---[0.02% 0.016252ms ] org.apache.catalina.connector.Request:getRemoteAddr() #613
        +---[0.00% 0.002572ms ] org.apache.catalina.connector.Request:getRemoteHost() #614
        +---[0.00% 0.001721ms ] org.apache.catalina.connector.Request:getScheme() #615
        +---[0.00% 0.001826ms ] org.apache.catalina.connector.Request:isSecure() #616
        +---[0.00% 0.001719ms ] org.apache.catalina.connector.Request:getServerName() #617
        +---[0.00% 0.002139ms ] org.apache.catalina.valves.RemoteIpValve:isChangeLocalName() #618
        +---[0.00% 0.004452ms ] org.apache.catalina.connector.Request:getServerPort() #619
        +---[0.00% 0.002919ms ] org.apache.catalina.connector.Request:getLocalPort() #620
        +---[0.00% 0.001888ms ] org.apache.catalina.connector.Request:getHeader() #621
        +---[0.00% 0.003649ms ] org.apache.catalina.connector.Request:getHeader() #622
        +---[0.00% 0.0024ms ] org.apache.catalina.connector.Request:getHeaders() #632
        +---[0.01% 0.011289ms ] org.apache.catalina.valves.RemoteIpValve:commaDelimitedListToStringArray() #640
        +---[0.00% 0.001824ms ] org.apache.catalina.connector.Request:setRemoteAddr() #667
        +---[0.00% 0.001824ms ] org.apache.catalina.connector.Request:getConnector() #668
        +---[0.00% 0.002051ms ] org.apache.catalina.connector.Connector:getEnableLookups() #668
        +---[0.00% 0.001576ms ] org.apache.catalina.connector.Request:setRemoteHost() #682
        +---[0.00% 0.001515ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #686
        +---[0.00% 0.00188ms ] org.apache.coyote.Request:getMimeHeaders() #686
        +---[0.00% 0.001862ms ] org.apache.tomcat.util.http.MimeHeaders:removeHeader() #686
        +---[0.00% 0.004026ms ] org.apache.tomcat.util.buf.StringUtils:join() #694
        +---[0.00% 0.001333ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #695
        +---[0.00% 0.001517ms ] org.apache.coyote.Request:getMimeHeaders() #695
        +---[0.00% 0.001971ms ] org.apache.tomcat.util.http.MimeHeaders:setValue() #695
        +---[0.00% 0.001817ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #695
        +---[0.00% 0.00168ms ] org.apache.catalina.connector.Request:getHeader() #700
        +---[0.00% 0.003129ms ] org.apache.catalina.valves.RemoteIpValve:isForwardedProtoHeaderValueSecure() #704
        +---[0.00% 0.001654ms ] org.apache.catalina.connector.Request:setSecure() #709
        +---[0.00% 0.001344ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #710
        +---[0.00% 0.002058ms ] org.apache.coyote.Request:scheme() #710
        +---[0.00% 0.001186ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #710
        +---[0.00% 0.002904ms ] org.apache.catalina.valves.RemoteIpValve:setPorts() #711
        +---[0.00% 0.001491ms ] org.apache.catalina.connector.Request:getHeader() #716
        +---[0.00% 0.003306ms ] org.apache.tomcat.util.http.parser.Host:parse() #719
        +---[0.00% 0.001296ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #725
        +---[0.00% 0.001337ms ] org.apache.coyote.Request:serverName() #725
        +---[0.00% 0.001271ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #725
        +---[0.00% 0.001253ms ] org.apache.catalina.valves.RemoteIpValve:isChangeLocalName() #726
        +---[0.00% 0.003314ms ] org.apache.catalina.connector.Request:setAttribute() #736
        +---[0.02% 0.015683ms ] org.apache.juli.logging.Log:isDebugEnabled() #738
        +---[0.00% 0.00135ms ] org.apache.catalina.connector.Request:getRemoteAddr() #756
        +---[0.00% 0.001719ms ] org.apache.catalina.connector.Request:setAttribute() #755
        +---[0.00% 0.001249ms ] org.apache.catalina.connector.Request:getRemoteAddr() #758
        +---[0.00% 0.001294ms ] org.apache.catalina.connector.Request:setAttribute() #757
        +---[0.00% 0.00128ms ] org.apache.catalina.connector.Request:getRemoteHost() #760
        +---[0.00% 0.001485ms ] org.apache.catalina.connector.Request:setAttribute() #759
        +---[0.00% 0.002571ms ] org.apache.catalina.connector.Request:getProtocol() #762
        +---[0.00% 0.001857ms ] org.apache.catalina.connector.Request:setAttribute() #761
        +---[0.01% 0.008126ms ] org.apache.catalina.connector.Request:getServerName() #764
        +---[0.00% 0.001972ms ] org.apache.catalina.connector.Request:setAttribute() #763
        +---[0.00% 0.001452ms ] org.apache.catalina.connector.Request:getServerPort() #766
        +---[0.00% 0.001782ms ] org.apache.catalina.connector.Request:setAttribute() #765
        +---[0.00% 0.001442ms ] org.apache.catalina.valves.RemoteIpValve:getNext() #769
        +---[99.65% 100.500217ms ] org.apache.catalina.Valve:invoke() #769
        +---[0.00% 0.002653ms ] org.apache.catalina.connector.Request:setRemoteAddr() #771
        +---[0.00% 0.001491ms ] org.apache.catalina.connector.Request:setRemoteHost() #772
        +---[0.00% 0.00171ms ] org.apache.catalina.connector.Request:setSecure() #773
        +---[0.00% 0.001662ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #774
        +---[0.00% 0.00229ms ] org.apache.coyote.Request:scheme() #774
        +---[0.00% 0.001822ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #774
        +---[0.00% 0.00142ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #775
        +---[0.00% 0.002272ms ] org.apache.coyote.Request:serverName() #775
        +---[0.00% 0.001255ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #775
        +---[0.00% 0.002824ms ] org.apache.catalina.valves.RemoteIpValve:isChangeLocalName() #776
        +---[0.00% 0.001433ms ] org.apache.catalina.connector.Request:setServerPort() #779
        +---[0.00% 0.001558ms ] org.apache.catalina.connector.Request:setLocalPort() #780
        +---[0.00% 0.001666ms ] org.apache.catalina.connector.Request:getCoyoteRequest() #782
        +---[0.00% 0.001526ms ] org.apache.coyote.Request:getMimeHeaders() #782
        +---[0.00% 0.002167ms ] org.apache.tomcat.util.http.MimeHeaders:removeHeader() #784
        +---[0.00% 0.002147ms ] org.apache.tomcat.util.http.MimeHeaders:setValue() #792
        `---[0.00% 0.001765ms ] org.apache.tomcat.util.buf.MessageBytes:setString() #792
```

对比代码发现dev环境的确执行了删除header的操作



仔细探究RemoteIpValve代码逻辑

和目前问题相关的大致功能为：解析X-Forwarded-for请求头，将其中的远端地址设置到RemoteAddr中

```
简单解释X-Forwarded-For的作用

例如真正的用户客户端是Client1，通过代理服务器proxy1，proxy2，到达服务器，在Tomcat中执行获取客户端地址的方法：request.getRemoteAddr，获得的IP地址是proxy2的，也就是负载均衡的地址；

而如果你想要获取Client1的地址，也是可以获取到的，就是通过X-Forwarded-For字段；
X-Forwarded-For:简称XFF头，它代表客户端，只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。
X-Forwarded-For内置在Http协议头中，刚刚的场景X-Forwarded-For取值为：client1, proxy1, proxy2
```

具体逻辑为：

对于列表中的每个ip，如果属于内网地址则跳过，否则将此ip设置为远程ip，停止循环

```java
//最近一跳代理地址
final String originalRemoteAddr = request.getRemoteAddr();
...
//X-Forwarded-For头信息
final String originalRemoteIpHeader = request.getHeader(remoteIpHeader);
//最近一跳代理地址是否为内网
// 内网正则判断为 Pattern internalProxies = Pattern.compile(
        "10\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}|" +
        "192\\.168\\.\\d{1,3}\\.\\d{1,3}|" +
        "169\\.254\\.\\d{1,3}\\.\\d{1,3}|" +
        "127\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}|" +
        "172\\.1[6-9]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
        "172\\.2[0-9]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
        "172\\.3[0-1]{1}\\.\\d{1,3}\\.\\d{1,3}|" +
        "0:0:0:0:0:0:0:1|::1");
//originalRemoteAddr 所有环境都为内网地址，返回true
boolean isInternal = internalProxies != null &&
        internalProxies.matcher(originalRemoteAddr).matches();
//trustedProxies为空
if (isInternal || (trustedProxies != null &&
        trustedProxies.matcher(originalRemoteAddr).matches())) {
    String remoteIp = null;
    Deque<String> proxiesHeaderValue = new LinkedList<>();
    StringBuilder concatRemoteIpHeaderValue = new StringBuilder();
 
    for (Enumeration<String> e = request.getHeaders(remoteIpHeader); e.hasMoreElements();) {
        if (concatRemoteIpHeaderValue.length() > 0) {
            concatRemoteIpHeaderValue.append(", ");
        }
 
        concatRemoteIpHeaderValue.append(e.nextElement());
    }
    //X-Forwarded-For内的地址转换为数组
    String[] remoteIpHeaderValue = commaDelimitedListToStringArray(concatRemoteIpHeaderValue.toString());
    int idx;
    if (!isInternal) {
        proxiesHeaderValue.addFirst(originalRemoteAddr);
    }
    //从最后的地址循环
    for (idx = remoteIpHeaderValue.length - 1; idx >= 0; idx--) {
        String currentRemoteIp = remoteIpHeaderValue[idx];
        remoteIp = currentRemoteIp;
        //如果是内网地址则跳过
        if (internalProxies !=null && internalProxies.matcher(currentRemoteIp).matches()) {
            
        //trustedProxies目前配置为空
        } else if (trustedProxies != null &&
                trustedProxies.matcher(currentRemoteIp).matches()) {
            proxiesHeaderValue.addFirst(currentRemoteIp);
        } else {
        //找到第一个不是内网地址的idx，但如果只有一个地址，则idx会变成负数
            idx--;
            break;
        }
    }
    //重新构建客户端地址的list，但idx必须不小于0
    LinkedList<String> newRemoteIpHeaderValue = new LinkedList<>();
    for (; idx >= 0; idx--) {
        String currentRemoteIp = remoteIpHeaderValue[idx];
        newRemoteIpHeaderValue.addFirst(currentRemoteIp);
    }
     
...
    //如果newRemoteIpHeaderValue为空则删除X-Forwarded-For
    if (newRemoteIpHeaderValue.size() == 0) {
         request.getCoyoteRequest().getMimeHeaders().removeHeader(remoteIpHeader);
    }
```

总结一下逻辑

```
X-Forwarded-For中的地址集合从后往前取，在至少有两个地址并且，最后的地址是内网地址的情况下不会删除X-Forwarded-For请求头
```

看一下dev、test、线上的数据

```
//test环境 172.31.0.203为内网地址
X-Forwarded-For: 203.187.160.86, 100.117.125.137, 172.31.0.203
//dev环境172.30.1.118为内网地址
X-Forwarded-For: 172.30.1.118
//线上环境 117.136.68.19为外网地址，但只有一个ip，没有代理的地址
X-Forwarded-For: 117.136.68.19
```

所以dev和线上会被删去请求头

### 处理

在wb-base-component-starter中添加公共配置

```
server:
    forward-headers-strategy: none
```

不加载tomcat的RemoteIpValve