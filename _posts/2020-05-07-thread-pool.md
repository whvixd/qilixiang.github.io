---
layout:     post
title:      ThreadPoolExecutor
subtitle:   
date:       2020-05-07
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - Java ThreadPoolExecutor
    
---
## What?
#### 1. 简介
线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。

> 本文主要介绍j.u.c.ThreadPoolExecutor线程池的使用及源码分析

#### 2. 成员
1. **corePoolSize**:核心线程数
2. **maximumPoolSize**:最大线程数
3. **keepAliveTime**:队列中任务等待执行时间
4. **unit**:队列中任务等待执行时间单位
5. **workQueue**:等待执行任务的队列
6. **threadFactory**:新线程的创建方式
7. **handler**:队列满了且大于最大线程数后任务的拒绝策略

> 这7个参数在本文后面源码分析中会详细介绍

---

## Why?
1. **降低资源**(利用重复机制，降低创建和销毁线程的次数)
2. **提高执行效率**(当任务到达时，不需要等待线程的创建就能立马执行)
3. **提高线程可管理性**(线程池是稀缺资源，使用线程池可以限制创建，充分利用系统资源，提高系统的稳定性，进行统一的分配，调优和监控)

> 若想合理使用线程池，必须了解线程池的核心参数

---

## How?

#### 1. Executors创建线程池

方法 | 说明
---|---
newFixedThreadPool(int nThreads)|创建固定线程数的线程池
newCachedThreadPool()|创建线程数不受上限的线程池，任何提交的任务都将立即执行
newSingleThreadExecutor()|创建一个线程的线程池
newScheduledThreadPool(int corePoolSize)|创建核心线程数为corePoolSize，可执行定时任务的线程池
newSingleThreadScheduledExecutor()|创建一个线程的定时线程池

> 不推荐，阿里巴巴开发手册:【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样
的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

**newFixedThreadPool()源码如下**

```java
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```
> 没有指定等待任务队列长度，可能会堆积大量的请求，从而导致 OOM。

**newCachedThreadPool()源码如下**

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
> 最大线程数为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM

#### 2. ThreadPoolExecutor核心参数详解
1. **corePoolSize**:创建真正的线程数，执行任务和等待队列中的任务，当线程数小于该值时，优先执行新建的任务
2. **maximumPoolSize**:线程池中运行的最大线程池
3. **keepAliveTime**:当线程数大于核心线程数时，多余线程等待新任务的最长时间，若超时，则线程退出
4. **unit**:队列任务等待时间的单位
5. **workQueue**:当线程数大于核心线程数时，用来缓存未执行的任务，队列会一直持有任务直到有线程开始执行它
6. **threadFactory**:通过工厂创建自定义名字的线程
7. **handler**:当线程数大于最大线程数且队列已满时，采取拒绝策略处理新任务

**常用堵塞队列**

堵塞队列 | 说明
---|---
ArrayBlockingQueue|基于数组结构的堵塞队列，可自定义队列长度
LinkedBlockingQueue|基于链表的堵塞队列，可自定义队列长度
SynchronousQueue|同步队列，该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直被阻塞

**ThreadPoolExecutor提供四种拒绝策略**

拒绝策略 | 说明
---|---
AbortPolicy|直接丢弃新任务，并抛出 RejectedExecutionException，线程池默认策略
DiscardPolicy|无任何处理，直接丢弃新任务
DiscardOldestPolicy|丢弃队首任务，并执行新任务
CallerRunsPolicy|若线程没有退出时，由当前线程执行新任务
