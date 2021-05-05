---
layout:     post
title:      Spring的事件机制
subtitle:   
date:       2021-05-05
author:     Static
header-img: 
catalog: true
tags:
    - Spring
    
---

### 1. What?

Spring的`ApplicationEventPublisher`拥有事件发布并且注册事件监听器的能力，拥有一套完整的事件发布与监听机制，类似于Guava的EventBus。

---

### 2. Why?

事件处理机制，是设计模式中的观察者模式（生产/消费者编程模型）的优雅实现，高类聚低耦合，降低业务代码的耦合度，支持异步操作。

---

### 3. How?

**核心类：**

a. 事件`ApplicationEvent`; 

b. 事件发布者`ApplicationEventPublisher`; 

c. 事件监听器`ApplicationListener`或注解`@EventListener`

---

**步骤：**

1.创建自定义的事件模型

```java
@Data
public class Something {
    // 自定义字段
    private String id;
    private String type;
    private String name;
    // ...
}
```

2.自定义事件继承`ApplicationEvent`

```java
public class SomethingRegisterEvent extends ApplicationEvent{
    public SomethingRegisterEvent(Something something) {
        super(something);
    }
}
```

3.创建事件发布者

```java
@Service
@Slf4j
public class EventService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void publishSomething(Something something) {
        publisher.publishEvent(new SomethingRegisterEvent(something));
        log.info("EventService->publishSomething publish success,something:{}", something);
    }
}
```

4.添加事件监听器

a.实现接口方式

```java
@Component
@Slf4j
public class SomethingAsyncEventListener implements ApplicationListener<SomethingRegisterEvent> {

    @Override
    // 通过自定义线程池实现异步
    @Async("commonExecutorPool")
    public void onApplicationEvent(@NonNull SomethingRegisterEvent somethingRegisterEvent) {
        Something something = (Something) somethingRegisterEvent.getSource();
        handlerSomething(something);
        log.info("SomethingAsyncEventListener->onApplicationEvent success");
    }

    private void handlerSomething(Something something) {
        log.info("SomethingAsyncEventListener->handler something:{}", something);
    }

}
```

b.通过注解实现

```java
@Component
@Slf4j
public class SomethingEventOneListener {
    @EventListener(value = SomethingRegisterEvent.class, condition = "#event.source.type=='opt'")
    public void firstHandler(SomethingRegisterEvent event) {
        log.info("SomethingEventOneListener->firstHandler success,event:{}", event);
    }

    @EventListener(value = SomethingRegisterEvent.class, condition = "#event.source.type=='opt'")
    // 通过自定义线程池实现异步
    @Async("commonExecutorPool")
    public void secondHandler(SomethingRegisterEvent event) {
        log.info("SomethingEventOneListener->secondHandler success,event:{}", event);
    }
}
```
> 可通过自定义状态机实现策略模式，线程池实现：[传送门](http://whvixd.com/2021/04/25/MDC/)

5.测试

```java
@SpringBootTest(classes = {SpringBootDemoApplication.class})
@RunWith(SpringRunner.class)
public class EventServiceTest {
    @Autowired
    private EventService eventService;

    @Test
    public void test() {
        Something something = new Something();
        something.setId("0");
        something.setName("test");
        something.setType("opt");
        eventService.publishSomething(something);

    }
}
```