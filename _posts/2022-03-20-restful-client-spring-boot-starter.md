---
layout:     post
title:      构建restful-client的SpringBoot启动器
subtitle:   
date:       2022-03-20
author:     Static
header-img: 
catalog: true
tags:
    - Spring
    
---

# 一、前言

## 1. 什么是spring-boot-starter

spring-boot-starter能够抛弃以前spring项目中繁杂的配置，将其统一集成进starter，应用者只需要在maven中引入starter依赖，SpringBoot就能自动扫描到要加载的信息并启动相应的默认配置。

## 2. 为什么使用spring-boot-starter

spring-boot-starter让我们摆脱了各种依赖库的处理和各种信息的困扰，减少不必要的重复代码的开发，也简化了很多繁琐的配置，让开发人员集中精力放在业务代码的开发工作上。


## 3. 常用的spring-boot-starter项目

1. spring-boot-starter-logging、spring-boot-starter-log4j、spring-boot-starter-log4j2，可快速集成和配置项目中的日志。

2. spring-boot-starter-web，可快速使用SpringMVC开发web应用，简化并开发一个Web项目，如果不想使用嵌入式的tomcat，那么可以引用spring-boot-starter-jetty或spring-boot-starter-undertow作为Web容器。

3. spring-boot-starter-jdbc，自动配置数据访问的基础设施，也可以使用mybatis-spring-boot-starter作为数据访问，还有像mybatis-plus-boot-starter、dynamic-datasource-spring-boot-starter之类的数据访问项目。

4. spring-boot-starter-aop，提供代码生成、动态代理、字节码增强等功能。

5. spring-boot-starter-actuator，提供应用的监控功能。

# 二、restful-client简介

> 之前写过一篇介绍restful-client的文章（[传送门](http://whvixd.com/2020/03/27/restful-client/)），这里就不做过多赘述了。

**1. 什么是restful-client**

restful-client是一款基于okhttp开发的restful风格的http客户端项目，其原理是利用动态代理开发便携式注解可快速开发http协议的客户端接口，相当于简化版的[retrofit](https://square.github.io/retrofit/)。


**2. 如何使用restful-client**

定义客户端接口，添加`@RequestMapping(path = "127.0.0.1:8080")`注解，再添加相关的get、post请求的方法，并添加`@RequestGet(path = "/hello/get")`之类的注解。


# 三、构建restful-client的SpringBoot版本

> 下面简单介绍下构建restful-client的starter项目的流程和使用

## 1. 初始化SpringBoot的脚手架项目

可使用[Spring官网的脚手架](http://start.spring.io/)构建，也可使用idea去构建SpringBoot项目

## 2. 添加相关依赖的jar包

restful-client-spring-boot-stater所依赖的包：


    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <optional>true</optional>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>${okhttp.version}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>${fastjson.version}</version>
        </dependency>
    </dependencies>


## 3. 添加自动配置类

```java
@Configuration
@EnableConfigurationProperties(RestfulClientProperties.class)
public class RestfulClientAutoConfiguration {
    private final RestfulClientProperties restfulClientProperties;

    @Autowired
    public RestfulClientAutoConfiguration(RestfulClientProperties restfulClientProperties) {
        this.restfulClientProperties = restfulClientProperties;
    }

    @Bean
    public RestfulClientActuator restfulClientActuator() {
        return new RestfulClientActuator(restfulClientProperties);
    }

    @Bean
    public RestfulClientDispatcher restfulClientDispatcher() {
        return new RestfulClientDispatcher();
    }

    @Bean
    public RestfulClientProxy restfulClientProxy() {
        return new RestfulClientProxy();
    }

}
```

## 4. 添加spring.factories

在resources目录下添加`META-INF/spring.factories`，让SpringBoot识别到自动配置类。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.github.whvixd.restful.client.spring.boot.autoconfigure.RestfulClientAutoConfiguration
```

如何识别spring.factories中的配置类？如何实例化配置类？
见spring源码中的`SpringFactoriesLoader#loadFactoryNames`方法和`SpringApplication#createSpringFactoriesInstances`


## 5. 添加RestfulClientFactoryBean

RestfulClientFactoryBean 是实现 Spring的FactoryBean，目的是对客户端接口进行动态代理

```java
public class RestfulClientFactoryBean<T> implements FactoryBean<T> {

    private final Class<T> clientType;

    @Autowired
    private RestfulClientProxy proxy;

    public RestfulClientFactoryBean(Class<T> clientType) {
        this.clientType = clientType;
    }

    @Override
    public T getObject() throws Exception {
        return proxy.wrapDynamicProxy(clientType);
    }

    @Override
    public Class<T> getObjectType() {
        return clientType;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

## 6. 利用@RestfulClientScan注解识别客户端接口

@RestfulClientScan需要在项目的SpringBoot启动类或配置类上标识restful-client的包路径，如下：

```java
@Configuration
@EnableAutoConfiguration
@RestfulClientScan(basePackages = "com.github.whvixd.restful.client.spring.boot.testclient")
static class RestfulClientScanConfiguration {
}
```

@RestfulClientScan的实现原理见`ClassPathRestfulClientScanner`、`RestfulClientScannerConfigurer`和`RestfulClientScannerRegister`

## 7. 如何使用restful-client的starter

### 1. 下载jar包

[进入页面](https://github.com/whvixd/restful-client-spring-boot-starter/releases)，下载jar包到本地（未发布到maven仓库）

### 2. 将下载的jar包添加到本地依赖中

```bash
mvn install:install-file -DgroupId=com.github.whvixd -DartifactId=restful-client-spring-boot-starter -Dversion=1.0.0 -Dpackaging=jar -Dfile=/Users/xxx/Downloads/restful-client-spring-boot-starter-1.0.0.jar
```

> 注意包路径，改成自己本地的路径

### 3. 本地spring-boot项目添加mvn依赖

     <dependency>
            <groupId>com.github.whvixd</groupId>
            <artifactId>restful-client-spring-boot-starter</artifactId>
            <version>1.0.0</version>
     </dependency>
     
     
### 4. spring-boot启动类添加`@RestfulClientScan`

```java
@SpringBootApplication
@RestfulClientScan(basePackages = "com.github.whvixd.restful.client.spring.boot.client")
public class RestfulClientApplication {
    public static void main(String[] args) {
            SpringApplication.run(RestfulClientApplication.class, args);
        }
    
}
```

### 5. 添加`client`接口

```java
@RequestMapping(path = "192.168.22.22:8888",message = "helloClient")
public interface HelloRestfulClient {
    @RequestGet(path = "/hello/get")
    String helloGet(@RequestHeader Map<String, String> headers);

    @RequestGet(path = "/hello/get/{type}/{id}")
    String helloGetPath(@RequestHeader Map<String, String> headers, @RequestPathParam Map<String, String> pathParam);

    @RequestGet(path = "/hello/get/{type}?id={id}&name={name}")
    String helloGetQuery(@RequestHeader Map<String, String> headers, @RequestPathParam Map<String, String> pathParam, @RequestQueryParam Map<String, String> queryParam);

    @RequestPost(path = "/hello/post")
    String helloPost(@RequestHeader Map<String, String> headers, @RequestBody Map<String, Object> body);

    @RequestPost(path = "/hello/post/serialize")
    RestfulClientAutoConfigurationTest.HelloPostRes helloPostSerialize(@RequestHeader Map<String, String> headers, @RequestBody RestfulClientAutoConfigurationTest.HelloPostBody body);
}
```


### 6. `client`的使用

```java
@Service
public class HelloService{
    @Autowired
    private HelloRestfulClient helloRestfulClient;
    
    public String helloGet(){
        Map<String,String> mockHeaders=new HashMap<>();
        // http远程调用
        return helloRestfulClient.helloGet(mockHeaders);
    }
} 
```

> 可参考测试代码：[RestfulClientAutoConfigurationTest](https://github.com/whvixd/restful-client-spring-boot-starter/blob/master/src/test/java/com/github/whvixd/restful/client/spring/boot/autoconfigure/RestfulClientAutoConfigurationTest.java)

> 源码链接：[https://github.com/whvixd/restful-client-spring-boot-starter](https://github.com/whvixd/restful-client-spring-boot-starter)