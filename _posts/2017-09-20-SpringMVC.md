---
layout:     post
title:      SpringMVC
subtitle:   总结SpringMVC
date:       2017-09-20
author:     Static
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - SpringMVC
---

# 初步认识SpringMVC

#### Why?
  为什么使用SpringMvc？
Lifecycle for overriding binding, validation, etc，易于同其它View框架（Tiles等）无缝集成，采用IOC便于测试。
它是一个典型的教科书式的mvc构架，而不像struts等都是变种或者不是完全基于mvc系统的框架，对于初学者或者想了解mvc的人来说我觉得 spring是最好的，它的实现就是教科书！第二它和tapestry一样是一个纯正的servlet系统，这也是它和tapestry相比 struts所具有的优势。而且框架本身有代码，看起来容易理解。

#### What?
  什么是SpringMvc？
Spring MVC属于SpringFrameWork的后续产品，已经融合在Spring Web Flow里面。Spring 框架提供了构建 Web 应用程序的全功能 MVC 模块。使用 Spring 可插入的 MVC 架构，从而在使用Spring进行WEB开发时，可以选择使用Spring的SpringMVC框架或集成其他MVC开发框架，如Struts1，Struts2等。

#### How?
  如何使用SpringMvc？
##### SpringMVC的流程图

![SpringMVC的流程图](img/clipboard.png)

##### Spring工作流程描述

1. 首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；
2. DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；
3. DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；
4. HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；
5. ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；
6. View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；
7.返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束

##### 具体的核心开发步骤：

1. DispatcherServlet在web.xml中的部署描述，从而拦截请求到Spring Web MVC
2. HandlerMapping的配置，从而将请求映射到处理器
3. HandlerAdapter的配置，从而支持多种类型的处理器
4. ViewResolver的配置，从而将逻辑视图名解析为具体视图技术
5.处理器（页面控制器）的配置，从而进行功能处理

##### 通过注解的方式：

> 需要通过处理器映射DefaultAnnotationHandlerMapping和处理器适配器AnnotationMethodHandlerAdapter来开启支持@Controller 和 @RequestMapping注解的处理器。

@Controller：用于标识是处理器类；<br>
@RequestMapping：请求到处理器功能方法的映射规则；<br>
@RequestParam：请求参数到处理器功能处理方法的方法参数上的绑定；<br>
@ModelAttribute：请求参数到命令对象的绑定；<br>
@SessionAttributes：用于声明session级别存储的属性，放置在处理器类上，通常列出模型属性（如
@ModelAttribute）对应的名称，则这些属性会透明的保存到session中；<br>
@InitBinder：自定义数据绑定注册支持，用于将请求参数转换到命令对象属性的对应类型。

##### DispatcherServlet主要用作职责调度工作，本身主要用于控制流程，主要职责如下：

1.文件上传解析，如果请求类型是multipart将通过MultipartResolver进行文件上传解析；
2.通过HandlerMapping，将请求映射到处理器（返回一个 HandlerExecutionChain ，它包括一个处理器、多个HandlerInterceptor拦截器）；
<!--[if !supportLists]-->3、 <!--[endif]-->通过HandlerAdapter支持多种类型的处理器( HandlerExecutionChain中的处理器)；
4.通过ViewResolver解析逻辑视图名到具体视图实现；
5.本地化解析；
6.渲染具体的视图等；
7.如果执行过程中遇到异常将交给HandlerExceptionResolver来解析。
