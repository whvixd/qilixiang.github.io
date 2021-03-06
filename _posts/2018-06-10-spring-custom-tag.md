---
layout:     post
title:      Spring自定义标签
subtitle:   
date:       2018-06-10
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - Spring
---

### 如何用Spring去自定义标签

> 主要分为五步：

1. 首先我们需要一个POJO，用来测试自定义标签的值传递

2. 创建xsd文件主要用来自定义标签和命名空间

3. 创建解析BeanDefinition类，直接扩展Spring的AbstractSingleBeanDefinitionParser抽象类，重写它的getBeanClass和doParse方法

4. 创建Handler类，扩展Spring的NamespaceHandlerSupport抽象类，重写init方法，为了注册自定义解析BeanDefinition

5. 添加spring.handlers和spring.schemas文件，spring.handlers用来指定NamespaceHandler；spring.schemas用来指定自定义的xsd文件

### 根据以上五步，我们来实现一个自定义标签

- **创建Course类**

```java
package com.github.whvixd.demo;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;

public interface Entity {
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    class Course {
        private String courseName;
        private double score;
    }
}

```

- **在项目的META-INF目录下添加course.xsd**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<schema xmlns="http://www.w3.org/2001/XMLSchema"
        targetNamespace="http://www.qilixiang1118.top/schema/course"
        elementFormDefault="qualified">

    <element name="course">
        <complexType>
            <attribute name="id" type="string"/>
            <attribute name="courseName" type="string"/>
            <attribute name="score" type="double"/>
        </complexType>
    </element>

</schema>

```

- **创建CourseBeanDefinitionParser和MyNamespaceHandler类**


> CourseBeanDefinitionParser.java

```java
package com.github.whvixd.demo.springDemo;

import com.github.whvixd.demo.Entity;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser;
import org.springframework.util.StringUtils;
import org.w3c.dom.Element;

/**
 * Created by wangzhx on 2018/6/10.
 */
public class CourseBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
    @Override
    protected Class<?> getBeanClass(Element element) {
        return Entity.Course.class;
    }

    @Override
    protected void doParse(Element element, BeanDefinitionBuilder builder) {
        String courseName = element.getAttribute("courseName");
        String score = element.getAttribute("score");

        if (StringUtils.hasLength(courseName)) {
            builder.addPropertyValue("courseName", courseName);
        }

        if (StringUtils.hasLength(score)) {
            builder.addPropertyValue("score", score);
        }
    }
}

```

---

> NamespaceHandler.java

```java
package com.github.whvixd.demo.springDemo;

import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

/**
 * Created by wangzhx on 2018/6/10.
 */
public class MyNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        registerBeanDefinitionParser("course", new CourseBeanDefinitionParser());
    }
}

```

- **在resources的META-INF目录下创建spring.handlers和spring.schemas**

> spring.handlers

```
http\://www.qilixiang1118.top/schema/course=com.github.whvixd.demo.springDemo.MyNamespaceHandler
```

> spring.schemas

```
http\://www.qilixiang1118.top/schema/course.xsd=META-INF/course.xsd
```

- **创建customBean.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mycourse="http://www.qilixiang1118.top/schema/course"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.qilixiang1118.top/schema/course
       http://www.qilixiang1118.top/schema/course.xsd">

    <mycourse:course id="chinese" courseName="语文" score="99.0"/>

</beans>
```

- **测试**

> SpringDemo.java

```java
package com.github.whvixd.testSpring;

import com.github.whvixd.demo.Entity;
import lombok.extern.slf4j.Slf4j;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

@Slf4j
public class SpringDemo {
    @Test
    public void myCourseBean(){

        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring/customBean.xml");
        Entity.Course course = (Entity.Course) context.getBean("chinese");
        System.out.println(course);

    }
}

```

> Run 打印Entity.Course(courseName=语文, score=99.0) ,成功！
