---
layout:     post
title:      SpringMVC项目中的注解
subtitle:   初学SpringMVC整理的一些知识点
date:       2017-09-24
author:     Static
header-img: img/post-bg-20170924.jpg
catalog: true
tags:
    - SpringMVC注解
---

## SpringMvc项目中常用的注解
### Dao层
#### @Repository:
Spring 自 2.0  版本开始，陆续引入了一些注解用于简化 Spring 的开发。@Repository注解便属于最先引入的一批，它用于将数据访问层 (DAO 层 ) 的类标识为 Spring Bean。具体只需将该注解标注在 DAO类上即可,同时，为了让 Spring 能够扫描类路径中的类并识别出 @Repository 注解，需要在 XML 配置文件中启用Bean 的自动扫描功能，这可以通过<context:component-scan/>实现。如下所示：

```
// 首先使用 @Repository 将 DAO 类声明为 Bean 
 package org.lanqiao.component.action;
 @Repository 
 public class StudentDao implements IStudentDao { …… } 

 // 其次，在 XML 配置文件中启动 Spring 的自动扫描功能
 <beans … > 
    ……
 <context:component-scan base-package="org.lanqiao.component">
 <!-- component下有dao、service和API -->
……
 </beans>
```
#### @Autowired 与 @Qualifier
@Autowired默认是按照类型装配注入的，如果想按照名称来转配注入，则需要结合@Qualifier一起使用;比如在Spring上下文中有几个JdbcTemplate的bean类型时，用@Qualifier("jdbcTemplate")来区别是哪个bean；

```
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
		<property name="dataSource" ref="dataSource"></property>
</bean>
```

---

### Service层

#### @Service
用于标注业务层组件，配置与@Repository相似；
#### @Transactional
##### 什么是事务？
通过一个例子来了解事务，比如日常生活中最常干的事：取钱。
比如你去ATM机取1000块钱，大体有两个步骤：首先输入密码金额，银行卡扣掉1000元钱；然后ATM出1000元钱。这两个步骤必须是要么都执行要么都不执行。如果银行卡扣除了1000块但是ATM出钱失败的话，你将会损失1000元；如果银行卡扣钱失败但是ATM却出了1000块，那么银行将损失1000元。所以，如果一个步骤成功另一个步骤失败对双方都不是好事，如果不管哪一个步骤失败了以后，整个取钱过程都能回滚，也就是完全取消所有操作的话，这对双方都是极好的，通过这个例子应该懂得事务是一个什么概念了吧。

##### 为什么用到事务？
事务用来解决类似问题的。事务是一系列的动作，它们综合在一起才是一个完整的工作单元，这些动作必须全部完成，如果有一个失败的话，那么事务就会回滚到最开始的状态，仿佛什么都没发生过一样。在企业级应用程序开发中，事务管理必不可少的技术，用来确保数据的完整性和一致性。 

##### 事务有四个特性：ACID


- 原子性（Atomicity）：事务是一个原子操作，由一系列动作组成。事务的原子性确保动作要么全部完成，要么完全不起作用。
- 一致性（Consistency）：一旦事务完成（不管成功还是失败），系统必须确保它所建模的业务处于一致的状态，而不会是部分完成部分失败。在现实中的数据不应该被破坏。
- 隔离性（Isolation）：可能有许多事务会同时处理相同的数据，因此每个事务都应该与其他事务隔离开来，防止数据损坏。
- 持久性（Durability）：一旦事务完成，无论发生什么系统错误，它的结果都不应该受到影响，这样就能从任何系统崩溃中恢复过来。通常情况下，事务的结果被写到持久化存储器中。

##### 基本事务属性
1. 是否读写
2. 传播行为
3. 隔离规则
4. 回滚规则
5. 事务超时

---

### Action层
#### @Controller
用于标注业务层组件；
#### @RequestMapping
RequestMapping是一个用来处理请求地址映射的注解，可用于类或方法上。用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。
##### RequestMapping注解有六个属性
1. value， method；

> value：指定请求的实际地址

```
@Controller
public class StudentAction {

	@Autowired
	IStudentService studentService;

	@RequestMapping(value = "students")
	public ModelAndView students() {
		ModelAndView mv = new ModelAndView();
		List<Student> students = studentService.getAll();
		mv.setViewName("studentList");
		mv.addObject("students", students);
		return mv;
	}
}
```
URL：http://localhost:8080/springMvcJdbc/students.action

> method：  指定请求的method类型， GET、POST、PUT、DELETE等；


```
@RequestMapping(value = "delete/{id}")
	public String delete(@PathVariable Integer id) {
	    ModelAndView mv = new ModelAndView();
	    Student stu = studentService.deleteById(id);
		mv.setViewName("deleteStudent");
		mv.addObject("student", stu);
		return "mv";
	}
```

2. consumes，produces；

> consumes： 指定处理请求的提交内容类型（Content-Type），例如application/json, text/html;

cousumes部分代码:
```
@Controller  
@RequestMapping(value = "/pets", 
                method = RequestMethod.POST,
                consumes="application/json")  
public void addStudent(@RequestBody Student stu, Model model) {      
    // implementation omitted  
}
```
方法仅处理request Content-Type为“application/json”类型的请求。


> produces:    指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；

produces部分代码:
```
@Controller  
    @RequestMapping(value = "/pets/{petId}",
                    method = RequestMethod.GET, 
                    produces="application/json")  
    @ResponseBody  
    public Pet getStudent(@PathVariable String stuId, Model model) {      
        // implementation omitted  
    }
```
方法仅处理request请求中Accept头中包含了"application/json"的请求，同时暗示了返回的内容类型为application/json;

3. params，headers；

> params： 指定request中必须包含某些参数值是，才让该方法处理。

```
@Controller  
    @RequestMapping("/owners/{ownerId}")  
    public class RelativePathUriTemplateController {  
      
      @RequestMapping(value = "/pets/{petId}", method = RequestMethod.GET, params="myParam=myValue")  
      public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {      
        // implementation omitted  
      }  
    }
```
仅处理请求中包含了名为“myParam”，值为“myValue”的请求；


> headers： 指定request中必须包含某些指定的header值，才能让该方法处理请求。

```
@Controller  
    @RequestMapping("/owners/{ownerId}")  
    public class RelativePathUriTemplateController {  
      
    @RequestMapping(value = "/pets", method = RequestMethod.GET, headers="Referer=http://www.qilixiang.com/")  
      public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {      
        // implementation omitted  
      }  
    }
```
 仅处理request的header中包含了指定“Refer”请求头和对应值为http://www.qilixiang.com 的请求
