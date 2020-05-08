---
layout:     post
title:      ThreadPoolExecutor源码
subtitle:   
date:       2020-05-08
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - Java ThreadPoolExecutor
    
---

## 1. 线程池使用

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(2, 10, 60,TimeUnit.SECONDS, new LinkedBlockingDeque<>(50), new ThreadPoolExecutor.CallerRunsPolicy());
executor.prestartAllCoreThreads();//启动所有核心线程
Runnable runnable = () -> {
    //do something
    System.out.println("run");
};
// 提交并执行任务
IntStream.range(0, 100).forEach(e -> {    
    executor.execute(runnable);
});

// 关闭线程池
if (!executor.isShutdown()) {
    executor.shutdown();
}

```
## 2. 线程池执行流程
<html>
    <img src="/img/ThreadPool/thread-pool-workflow.jpg" width="300" height="560" /> 
</html>

## 3. 源码分析

**1. [线程池的构造器参数介绍](http://whvixd.com/2020/05/07/thread-pool/)**

**2. 提交任务方法**

```java
    public void execute(Runnable command) {
        // 若任务为空直接异常
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        // 若当前工作线程数<核心线程数，则调用addWorker创建线程并执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 若大于corePoolSize，则任务入队列中等待
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次检测线程状态，若不是Running且成功从队列中删除，则任务由拒绝策略处理
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 若线程数空的，则新建一个线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 队列满了且线程数大于最大线程数，则任务由拒绝策略处理
        else if (!addWorker(command, false))
            reject(command);
    }
```

**3. 添加线程方法**

```java
   private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 线程是否关闭，
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 非核心线程，若线程数大于最大线程数，则失败，返回false
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 若cas修改成功，跳出外层循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 若失败，再次检测
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        // 线程是否执行成功标识
        boolean workerStarted = false;
        // 是否添加线程成功标识
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 新建线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 添加到set中
                        workers.add(w);
                        int s = workers.size();
                        // 数线程数量
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 添加线程成功
                        workerAdded = true;
                    }
                } finally {
                    // 释放锁资源
                    mainLock.unlock();
                }
                // 若添加线程成功，则执行任务
                if (workerAdded) {
                    // *调用 Worker.run()
                    t.start();
                    // 线程执行成功
                    workerStarted = true;
                }
            }
        } finally {
            // 若线程执行失败，则中set中删除线程
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

**4. Worker.run()**

Worker继承关系

<html>
    <img src="/img/ThreadPool/Worker-diagram.jpg"/> 
</html>

> Worker实现了Runnable,所以 addWorker()中执行线程最终调用的是Worker的run方法

  runWorker()

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 若核心线程的任务为空，则中队列中获取，*getTask()
            while (task != null || (task = getTask()) != null) {
              // 加锁
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    // 完成的任务数量
                    w.completedTasks++;
                    // 释放锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```

**5. getTask()**

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            // 若大于核心线程数，则超时
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 如果超时，poll时没有任务，则一直堵塞，若未超时则直接获取队头任务
                // 否则一直阻塞下去
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

**总结**
执行任务过程:ThreadPoolExecutor$execute -> ThreadPoolExecutor$addWorker -> Thead$start -> Worker$run() -> Worker$runWorker -> ThreadPoolExecutor$getTask

getTask自旋尝试获取任务并返回，若队列中没有任务则堵塞当前线程，runWorker自旋执行任务