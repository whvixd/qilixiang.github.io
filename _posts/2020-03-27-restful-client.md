---
layout:     post
title:      restful-client
subtitle:   rpc
date:       2020-03-27
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

项目源码:[https://github.com/whvixd/restful-client](https://github.com/whvixd/restful-client)


#### 如何使用？

##### 1. 添加Maven依赖

```
<groupId>com.github.whvixd</groupId>
<artifactId>restful-client</artifactId>
<version>1.0-SNAPSHOT</version>
```

##### 2. 添加spring bean配置:

> restful-client.xml

```xml
    <bean id="requestInvokeTest" class="com.github.restful.client.core.spring.RequestProxyFactoryBean"
              p:clientType="com.github.restful.client.core.RequestInvokeTest"/>
```

##### 3. 添加http client 接口，包含接口的请求头、请求体和响应体等，如下:

```groovy
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

##### 1. **流程图:**

<html>
    <img src="/img/rpc/restful-client.jpg" width="500" height="800" /> 
</html>


#### 2. 项目结构

<html>
    <img src="/img/rpc/structure.jpg" width="400" height="400" /> 
</html>

#### 3. 具体代码

- **RequestProxyFactoryBean** 继承 Spring FactoryBean，对调用方的接口或类进行代理

```java
@Data
public class RequestProxyFactoryBean<T> implements FactoryBean<T> {

    private Class<T> clientType;

    @Override
    public T getObject() throws Exception {
        return RequestProxy.INSTANCE.invoke(clientType);
    }

    @Override
    public Class<T> getObjectType() {
        return clientType;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

- **RequestProxy** 根据调用方的类型进行判断，若是接口则用Java带态代理，若是类则用Cglib动态代理；解析接口入参为RequestParam，RequestParam.getRequestParam(method, args)

```java
@Slf4j
@SuppressWarnings("all")
public enum RequestProxy {
    //单例模式
    INSTANCE;
    private RequestHandler requestHandler;
    
    private RequestProxy() {
        requestHandler = new RequestHandler();
    }

    // 判断调用方的类型
    public <T> T invoke(Class<T> clientType) {
        return clientType.isInterface()
                ? getProxyByInterface(clientType) : getProxyByClazz(clientType);
    }

    // 获取Java带态代理
    private <T> T getProxyByInterface(Class<T> clientType) {
        return Reflection.newProxy(clientType, (proxy, method, args) ->
                requestHandler.doInvoke(RequestParam.getRequestParam(method, args)));
    }

    // 获取Cglib动态代理
    private <T> T getProxyByClazz(Class<T> clientType) {
        return (T) Enhancer.create(clientType, (MethodInterceptor) (proxy, method, args, methodProxy) ->
                requestHandler.doInvoke(RequestParam.getRequestParam(method, args)));
    }

    private <T> T getJdkClientProxy(Class<T> clientType) {
        return (T) Proxy.newProxyInstance(clientType.getClassLoader(), new Class[]{clientType},
                (proxy, method, args) -> requestHandler.doInvoke(RequestParam.getRequestParam(method, args)));
    }


}
```

- **RequestHandler**，编码、解码处理器，支持自定义化，若未实现，则使用默认处理器DefaultCoderHandler；根据http method调用OkHttp

```java
@Slf4j
public class RequestHandler<T> {

    private OkHttpProxy okHttpProxy;

    public RequestHandler() {
        okHttpProxy = new OkHttpProxy();
    }

    public T doInvoke(RequestParam requestParam) {
        //编码、解码处理器，支持自定义化，若未实现，则使用默认处理器DefaultCoderHandler
        CoderHandler coderHandler = requestParam.getCoderHandler();
        RequestType requestType = requestParam.getRequestType();
        String url = requestParam.getUrl();
        Map<String, String> headers = requestParam.getHeaders();
        Object body = requestParam.getBody();
        //编码
        byte[] encode = coderHandler.encode(body);

        //解码
        T result = coderHandler.decode(getResult(requestType, url, headers, encode), requestParam.getResultType());
        log.info("requestType:{},url:{},headers:{},body:{},result:{}", requestType, url, headers, body, result);
        return result;

    }

    //根据http method调用OkHttp
    private byte[] getResult(RequestType requestType, String url, Map<String, String> headers, byte[] encode) {
        switch (requestType) {
            case Get:
                return okHttpProxy.doGet(url, headers);
            case Post:
                return okHttpProxy.doPost(url, headers, encode);
            case Put:
                return okHttpProxy.doPut(url, headers, encode);
            case Delete:
                return okHttpProxy.doDelete(url, headers, encode);
            default:
                throw new ServerException(String.format("暂不支持 %s 请求方式", requestType));
        }

    }
}
```

- **RequestInvokeTest** Spock单元测试

```groovy
class RequestInvokeTest extends Configuration {

    @Autowired
    HelloClient helloClient

    @RequestMapping(path = "127.0.0.1:8080", coder = HelloCoderHandler.class)
    interface HelloClient {
        @RequestGet(path = "/hello/get")
        String helloGet(@RequestHeader Map<String, String> headers)

        @RequestPost(path = "/hello/post")
        String helloPost(@RequestHeader Map<String, String> headers, @RequestBody Map<String, Object> body)
    }

    def "hello get"() {
        when:
        def result = helloClient.helloGet(Maps.newHashMap())
        then:
        result == excepted
        where:
        _ || excepted
        _ || "{\"message\":\"Hello Get\"}"
    }

    def "hello post"() {
        when:
        def result = helloClient.helloPost((Maps.newHashMap()), Maps.newHashMap())
        then:
        result == excepted
        where:
        _ || excepted
        _ || "{\"message\":\"Hello Post\"}"
    }

}

@ContextConfiguration(locations = "classpath:spring/restful-client.xml")
class Configuration extends BaseTest{
    //启动测试服务 @see Spark测试
    def setupSpec(){
        port(8080)
        get("/hello/get", { req, res -> "{\"message\":\"Hello Get\"}" })
        post("/hello/post", { req, res ->
            def body = req.body()
            print(body)
            return "{\"message\":\"Hello Post\"}"
        })
    }

    def cleanupSpec(){
        stop()
    }

}

```

---

> **项目源码:[https://github.com/whvixd/restful-client](https://github.com/whvixd/restful-client)**

> Spock单元测试可参考 [http://whvixd.com/2020/03/26/spock/](http://whvixd.com/2020/03/26/spock/)

> Spark可参考 [http://sparkjava.com/](http://sparkjava.com/)