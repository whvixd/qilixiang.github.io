---
layout:     post
title:      本地启动Tomcat
subtitle:   
date:       2021-05-29
author:     Static
header-img: 
catalog: true
tags:
    - Tomcat
    
---

> 本次介绍如何在本地将tomcat导入idea中，并启动起来。

## 1. 打开idea直接导入tomcat 

> 建议Fork，tomcat源码地址：[传送门](https://github.com/apache/tomcat)；tomcat git：`https://github.com/apache/tomcat.git`

**如下图：**

<html>
    <img src="/img/tool/import_tomcat_1.png" width="700" height="700" /> 
</html>

> 需切换版本，可执行 `git checkout 8.5.30`，切换到该版本后，再创建并切换到自己的分支上 `git checkout -b 8.5.30_local`

## 2. 在根目录添加home文件，与java同级，home目录添加`lib`、`logs`、`work` 文件


## 3. 将 `java` mark as `Sources Root`； `test` mark as `Test Sources Root`

> 右键 `java` >> Mark Directory as >> Sources Root ; 右键 `test` >> Mark Directory as >> Test Sources Root 

## 4. 根目录添加 `pom.xml` 文件

> 下载tomcat依赖的jar包

```xml
<?xml version="1.0" encoding="utf-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>Tomcat8.5.30</artifactId>
    <name>Tomcat8.5.30</name>
    <version>8.5.30</version>
    <build>
        <finalName>Tomcat8.5.30</finalName>
        <sourceDirectory>java</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>test</directory>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.easymock</groupId>
            <artifactId>easymock</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.7.0</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.5.1</version>
        </dependency>
    </dependencies>
</project>
```

## 5. 将 `conf` 和 `webapps` 移到 home目录下，并删除 `webapps` 下的`examples`文件

## 6. org.apache.catalina.startup.ContextConfig.configureStart 添加JSP解析器

```
    protected synchronized void configureStart() {
        // Called from StandardContext.start()

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("contextConfig.start"));
        }

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("contextConfig.xmlSettings",
                    context.getName(),
                    Boolean.valueOf(context.getXmlValidation()),
                    Boolean.valueOf(context.getXmlNamespaceAware())));
        }

        webConfig();

        // 解析JSP
        context.addServletContainerInitializer(new JasperInitializer(), null);

        if (!context.getIgnoreAnnotations()) {
            applicationAnnotationsConfig();
        }
        if (ok) {
            validateSecurityRoles();
        }

        // Configure an authenticator if we need one
        if (ok) {
            authenticatorConfig();
        }

        // Dump the contents of this pipeline if requested
        if (log.isDebugEnabled()) {
            log.debug("Pipeline Configuration:");
            Pipeline pipeline = context.getPipeline();
            Valve valves[] = null;
            if (pipeline != null) {
                valves = pipeline.getValves();
            }
            if (valves != null) {
                for (Valve valve : valves) {
                    log.debug("  " + valve.getClass().getName());
                }
            }
        }

        // Make our application available if no problems were encountered
        if (ok) {
            context.setConfigured(true);
        } else {
            log.error(sm.getString("contextConfig.unavailable"));
            context.setConfigured(false);
        }

    }
```

## 7. 将test 下的 util.TestCookieFilter 代码注释掉

## 8. 添加启动程序，并编译

```
Main class: org.apache.catalina.startup.Bootstrap

VM options: -Dcatalina.home="/Users/xx/Documents/workspace/idea/tomcat/home" -Dfile.encoding=UTF-8

working dirtory: /Users/xx/Documents/workspace/idea/tomcat
```

<html>
    <img src="/img/tool/tomcat_2.png" width="700" height="700" /> 
</html>

## 9. 启动成功

> 浏览器中输入 `http://localhost:8080/` ，进入tomcat首页，即启动成功

<html>
    <img src="/img/tool/tomcat_3.png" width="700" height="700" /> 
</html>


