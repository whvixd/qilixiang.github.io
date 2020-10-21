---
layout:     post
title:      如何将spring-framework导入idea中
subtitle:   
date:       2020-06-10
author:     Static
header-img: 
catalog: true
tags:
    - Spring
    
---

> Java开发基本上都会用Spring框架，但是绝大数只是在用上，不知道原理，这里记录下我在导入Spring源码出现的问题及解决方法

### 1. 环境

a. jdk8+

b. gradle2.5

c. idea-2017.2


### 2. 步骤

> spring-framework项目下 import-into-idea.md 其实已经介绍了导入idea步骤及常见问题

git地址:[链接](https://github.com/spring-projects/spring-framework)

> 可以fork之后，clone代码，自己拉个分支，调试代码


_Within your locally cloned spring-framework working directory:_

1. Pre-compile `spring-oxm` with `./gradlew cleanIdea :spring-oxm:compileTestJava`
2. Import into IDEA (File->import project->import from external model->Gradle)
3. Set the Project JDK as appropriate (1.8+)
4. Exclude the `spring-aspects` module (Go to File->Project Structure->Modules)
5. Code away

### 3. Known issues

1. `spring-oxm` should be pre-compiled since it's using repackaged dependencies (see *RepackJar tasks)
2. `spring-aspects` does not compile out of the box due to references to aspect types unknown to IDEA.
See https://youtrack.jetbrains.com/issue/IDEA-64446 for details. In the meantime, the 'spring-aspects'
should be excluded from the overall project to avoid compilation errors.
3. While all JUnit tests pass from the command line with Gradle, many will fail when run from IDEA.
Resolving this is a work in progress. If attempting to run all JUnit tests from within IDEA, you will
likely need to set the following VM options to avoid out of memory errors:
    -XX:MaxPermSize=2048m -Xmx2048m -XX:MaxHeapSize=2048m

### 4. Tips

In any case, please do not check in your own generated .iml, .ipr, or .iws files.
You'll notice these files are already intentionally in .gitignore. The same policy goes for eclipse metadata.

### 5. FAQ

Q. What about IDEA's own [Gradle support](https://confluence.jetbrains.net/display/IDEADEV/Gradle+integration)?

A. Keep an eye on https://youtrack.jetbrains.com/issue/IDEA-53476

### 6. 问题记录

#### 1. Plugin with id 'sonar-runner' not found. Open File

<html>
    <img src="/img/tool/import-spring-idea-q1.jpg" width="700" height="700" /> 
</html>

> gradle版本号问题，直接用idea推荐的版本，gradle-2.5 ，我的idea是 2017.2，问题解决

#### 2. 移除spring-aspects模块

#### 3. 编译Spring项目，提示jar包不存在

> 见3.1，运行 右侧gradle工具 spring-core -> Tasks -> other -> cglibRepackJar/objenesisRepackJar 两个命令

<html>
    <img src="/img/tool/import-spring-idea-q2.jpg" width="400" height="700" /> 
</html>

#### 4. 再次编译Spring项目提示 `Error:(19, 49) java: 找不到符号`

```
Error:(19, 49) java: 找不到符号
  符号:   类 AnnotationBeanConfigurerAspect
  位置: 程序包 org.springframework.beans.factory.aspectj
```

<html>
    <img src="/img/tool/import-spring-idea-q3.jpg" width="400" height="700" /> 
</html>

> 见3.2，将`spring-aspects`模块剔除掉，再次编译通过

#### 5. 再次编译Spring项目提示

<html>
    <img src="/img/tool/impor_spring_20201021_1.png" width="500" height="500" /> 
</html>

> spring是aspectj切面编程

**解决：**

<html>
    <img src="/img/tool/impor_spring_20201021_2.png" width="500" height="500" /> 
</html>


#### 6. 开启Spring源码之旅

> Ioc、AOP原理？请求调用到具体的业务代码是流程是什么样的？