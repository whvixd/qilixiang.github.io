---
layout:     post
title:      restful client
subtitle:   rpc
date:       2020-03-26
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - rpc 
    
---

> 实现HTTP Client调用工具，先介绍什么是RPC

### RPC定义

RPC（Remote Procedure Call Protocol）远程过程调用协议。一个通俗的描述是：客户端在不知道调用细节的情况下，调用存在于远程计算机上的某个对象，就像调用本地应用程序中的对象一样。

Client：RPC协议的调用方，对远程服务进行通讯的调用方。

可参考 [https://www.jianshu.com/p/2accc2840a1b](https://www.jianshu.com/p/2accc2840a1b)

**常用RPC框架**:[dubbo](http://dubbo.apache.org/en-us/)、[gRPC](https://www.grpc.io/)、[thrift](http://thrift.apache.org/)等

**HTTP client框架**:[okhttp](https://square.github.io/okhttp/)、[httpclient](http://hc.apache.org/httpclient-3.x/)等

---

### 进入正题
> 利用spring FactoryBean、动态代理和okhttp实现http client

源码地址:[https://github.com/whvixd/restful-client]([https://github.com/whvixd/restful-client)


#### 如何使用？
1. 添加Maven依赖

```
<groupId>com.github.whvixd</groupId>
<artifactId>restful-client</artifactId>
<version>1.0-SNAPSHOT</version>
```

2. 添加spring bean配置:

> restful-client.xml

```
    <bean id="requestInvokeTest" class="com.github.restful.client.core.spring.RequestProxyFactoryBean"
              p:clientType="com.github.restful.client.core.RequestInvokeTest"/>
```

3. 添加http client 接口，包含接口的请求头、请求体和响应体等，如下:

```
    @RequestMapping(path = "127.0.0.1:8080", coder = HelloCoderHandler.class)
    interface HelloClient {
        @RequestGet(path = "/hello/get")
        String helloGet(@RequestHeader Map<String, String> headers)

        @RequestPost(path = "/hello/post")
        String helloPost(@RequestHeader Map<String, String> headers, @RequestBody Map<String, Object> body)
    }
    
```

> 具体可参考项目中的RequestInvokeTest.groovy的使用

#### 如何实现？

**流程图:**

![流程图](/img/rpc/restful-client.jpg)

