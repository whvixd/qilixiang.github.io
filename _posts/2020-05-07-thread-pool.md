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
1. corePoolSize:核心线程数
2. maximumPoolSize:最大线程数
3. keepAliveTime:队列中任务等待执行时间
4. unit:队列中任务等待执行时间单位
5. workQueue:等待执行任务的队列
6. threadFactory:新线程的创建方式
7. handler:队列满了且大于最大线程数后任务的拒绝策略

> 这7个参数在本文后面源码分析中会详细介绍

## Why?
1. **降低资源**(利用重复机制，降低创建和销毁线程的次数)
2. **提高执行效率**(当任务到达时，不需要等待线程的创建就能立马执行)
3. **提高线程可管理性**(线程池是稀缺资源，使用线程池可以限制创建，充分利用系统资源，提高系统的稳定性，进行统一的分配，调优和监控)

> 若想合理使用线程池，必须了解线程池的核心参数
