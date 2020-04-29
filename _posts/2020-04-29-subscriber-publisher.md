---
layout:     post
title:      subscriber-publisher
subtitle:   subscriber-publisher
date:       2020-04-29
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - subscriber-publisher 
    
---

> 参考 [apache eventbus](https://camel.apache.org/components/latest/guava-eventbus-component.html)，实现**订阅者-发布者模式**

---

### what?

#### 1. 简介
发布者-订阅者实现了观察者模式，消息通知负责人通过中间商去注册/注销观察者，最后由消息通知负责人给观察者发布消息。

#### 2. 成员

**Agent**:代理商(发布者)，负责注册订阅者，消息的发布，具体逻辑实现类

**@Subscribe**:标识订阅者

**@Prior**:在同一类型阅读者中(方法的入参)的vip

**SubscriberMessage**:阅读者信息

**Report**:具体推送的消息

---

### why?

#### 优点:
1. 解耦了事件的发布者和订阅者，专人做专事
2. 简化了代码，业务逻辑更清晰
3. 可定制化属性，比如线程，异步化，同步化

---

### how?

#### 1. 流程图:

<html>
    <img src="/img/SubscriberPublisher/subscriber-publisher.jpg" width="700" height="650" /> 
</html>

#### 2. 使用:

```java
public class Subscriber {

    @Subscribe
    @Prior
    public void toDo(String action){
        System.out.println("I am Subscriber");
    }

}

// 同步推送消息
Agent syncAgent = new Agent();        // 新建代理人
syncAgent.register(new Subscriber()); // 注册订阅者
syncAgent.push(new Report("I come!"));// 发送消息

// 异步推送消息
Agent syncAgent = new Agent(true);        // 新建代理人
syncAgent.register(new Subscriber()); // 注册订阅者
syncAgent.push(new Report("I come!"));// 发送消息
```

#### 3. 具体代码实现:

- **注解**:

```java
/**
 * Created by wangzhx on 2019/7/14.
 * 同一类型下的优先排序
 */
@Target(value = {ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Prior {
    String message() default "";

    int rank() default 0;

}

/**
 * Created by wangzhx on 2019/6/25.
 * 订阅者
 */
@Target(value = {ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Subscribe {
}
```

- **Agent**:

```java
/**
 * Created by wangzhx on 2019/7/10.
 * 中介
 * 1. 注册订阅者
 * 2. 给订阅者推消息
 */
public class Agent {
    /**
     * 存储订阅者方法的信息
     * 储存格式:{className:SubscriberMessages}
     * <p>
     * 容器
     */
    private Map<String, List<SubscriberMessage>> container;
    /**
     * vip容器
     */
    private Map<String, List<SubscriberMessage>> priorContainer;
    /**
     * 线程池，可自定义实现
     */
    private ThreadPoolExecutor executor;

    /**
     * 是否异步
     */
    private boolean isAsync;


    public Agent() {
    }

    public Agent(boolean isAsync) {
        this.isAsync = isAsync;
    }

    /**
     * 注册订阅者
     */
    public void register(Object subscriber) {
        if (subscriber == null) {
            return;
        }
        setContainer(subscriber);
    }

    /**
     * 发送消息给订阅者
     */
    public void push(Report report) {
        if (report == null) {
            return;
        }
   
        execute(report.getClazzName(), report);
    }

    /**
     * 若类添加了@Prior，则添加@Subscribe的所有方法都是优先级较高
     * 若方法添加了@Prior，则添加@Subscribe的方法是优先级较高
     */
    private void setContainer(Object subscriber) {
        Class<?> clazz = subscriber.getClass();
        Prior clazzPrior = clazz.getDeclaredAnnotation(Prior.class);
        Method[] declaredMethods = clazz.getDeclaredMethods();
        if (declaredMethods.length != 0) {
            for (Method method : declaredMethods) {
                Subscribe subscribe = method.getAnnotation(Subscribe.class);
                if (subscribe != null) {
                    SubscriberMessage message = SubscriberMessage.builder().
                            methodMessage(method).
                            subscriber(subscriber).
                            clazz(clazz);
                    if (clazzPrior != null) {
                        initContainer(getKey(method), message, clazzPrior);
                    } else {
                        Prior methodPrior = method.getAnnotation(Prior.class);
                        initContainer(getKey(method), message, methodPrior);
                    }
                }
            }
        }
    }

    /**
     * 初始化容器
     */
    protected void initContainer(String key, SubscriberMessage message, Prior prior) {
        if (key == null || message == null) {
            return;
        }
        if (prior != null) {
            if (priorContainer == null) {
                priorContainer = new ConcurrentHashMap<>();
            }
            setMessages(key, message, getMessages(key, priorContainer), priorContainer);
        } else {
            if (container == null) {
                container = new ConcurrentHashMap<>();
            }
            setMessages(key, message, getMessages(key, container), container);
        }
    }

    /**
     * 获取订阅者消息
     */
    private List<SubscriberMessage> getMessages(String key, Map<String, List<SubscriberMessage>> container) {
        if (container == null) {
            container = new ConcurrentHashMap<>();
        }
        return container.get(key);
    }

    private void setMessages(String key,
                             SubscriberMessage message,
                             List<SubscriberMessage> messages,
                             Map<String, List<SubscriberMessage>> container) {
        if (messages != null) {
            messages.add(message);
        } else {
            messages = new ArrayList<>();
            messages.add(message);
            container.put(key, messages);
        }
    }

    /**
     * 初始化线程池
     */
    private void initThreadPool() {
        if (Objects.isNull(executor)) {
            ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 10, TimeUnit.SECONDS, new LinkedBlockingDeque<>());
            setExecutor(executor);
        }
    }

    /**
     * 获取容器的key(订阅者入参类型)
     */
    private String getKey(Method method) {
        if (method == null || method.getParameterTypes().length == 0) {
            return null;
        }
        return method.getParameterTypes()[0].getSimpleName();
    }

    /**
     * 推送消息，订阅者消费
     */
    private void execute(String containerKey, Report report) {
        // 先推送给vip
        doExecute(containerKey, report, priorContainer);
        // 再推送给普通用户
        doExecute(containerKey, report, container);
    }

    /**
     * 具体执行
     * 1. 若是异步，则初始化线程池执行
     * 2. 若是同步，则同步执行
     */
    private void doExecute(String containerKey, Report report, Map<String, List<SubscriberMessage>> container) {
        if (isAsync) {
            initThreadPool();
            if (checkNullMap(container) && checkNullList(container.get(containerKey))) {
                container.get(containerKey).forEach(message -> executor.execute(
                        InvokeTask.newInstance(() -> message.invoke(report.getContent()))));
            }
        } else {
            if (checkNullMap(container) && checkNullList(container.get(containerKey))) {
                container.get(containerKey).forEach(message -> message.invoke(report.getContent()));
            }
        }
    }

    public boolean checkNullList(List<?> messages) {
        return messages != null && messages.size() != 0;
    }

    public boolean checkNullMap(Map<?, ?> container) {
        return container != null && container.size() != 0;
    }

    public void setExecutor(ThreadPoolExecutor executor) {
        this.executor = executor;
    }

}
```

- **Report**:

```java
/**
 * Created by wangzhx on 2019/7/10.
 * 负责装载消息给订阅者
 */
@AllArgsConstructor
@Data
public class Report {
    private Object content;

    public String getClazzName() {
        if (content != null) {
            return content.getClass().getSimpleName();
        }
        return null;
    }
}
```

- **SubscriberMessage**:

```java
/**
 * 订阅者信息，维度到方法
 */
@Data
public class SubscriberMessage {
    // 订阅者
    private Object subscriber;
    // 阅读者的类
    private Class clazz;
    // 阅读者方法
    private Method method;
    private int parameterCount = 1;
    // 方法入参类型
    private Class<?> parameterType;
    private boolean isPrior;

    private SubscriberMessage() {
    }

    /**
     * 入参暂支持一个
     */
    public void setMethodMessage(Method method) {
        setMethod(method);
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (parameterTypes == null) {
            setParameterCount(0);
            throw new RuntimeException();
        }
        setParameterType(parameterTypes[0]);
    }

    /**
     * 构造者模式
     */
    public static SubscriberMessage builder() {
        return new SubscriberMessage();
    }

    public SubscriberMessage methodMessage(Method method) {
        setMethodMessage(method);
        return this;
    }

    public SubscriberMessage clazz(Class clazz) {
        setClazz(clazz);
        return this;
    }

    public SubscriberMessage subscriber(Object subscriber) {
        setSubscriber(subscriber);
        return this;
    }

    /**
     * 订阅者消费
     */
    void invoke(Object arg) {
        try {
            method.invoke(subscriber, arg);
        } catch (IllegalAccessException | InvocationTargetException e) {
            throw new RuntimeException();
        }
    }

}
```
