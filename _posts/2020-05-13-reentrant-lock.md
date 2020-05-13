---
layout:     post
title:      ReentrantLock源码分析
subtitle:   ReentrantLock
date:       2020-05-13
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - Java Lock
    
---

## 1. What?
#### 1.1 原子操作
> 前提:Java语言最终也是也汇编的方式执行，所以从汇编的角度理解对变量的赋值操作过程:**读取 -> 更新 -> 写入**,在多线程下,赋值操作是不安全的。

不可中断的一个或一系列操作，所以jdk提供**ReentrantLock**，在多线程的情况下，让线程并行操作改为高效的串行操作。

---

## 2. Why?
为了防止在多线程的情况下，对字段修改操作时，其他线程无法获取准确的值。

---

## 3. How?
#### 3.1 测试

> 首先测试在多线程中的自增操作是否真正的自增

<html>
    <img src="/img/reentrant-lock/code_demo.jpg" width="400" height="400" /> 
</html>

> 为了更清楚的看到效果，自增时线程堵塞10毫秒，发现在100个线程对同一个count自增时，线程执行无规律，最终没有达到预期的值(100);

#### 3.2 原子锁的使用

<html>
    <img src="/img/reentrant-lock/code_lock_demo.jpg" width="400" height="400" /> 
</html>

> 对自增代码块加锁后，线程执行有序，最终实现了在多线程的情况下的自增操作

#### 3.3 源码分析
>AbstractQueuedSynchronizer简称AQS

**非公平锁加锁的调用链**

ReentrantLock$lock -> ReentrantLock.Sync.NonfairSync$lock -> AQS$acquire -> AQS$tryAcquire && AQS$addWaiter && AQS$acquireQueued

> 原子锁的默认是非公平锁，大概过程就是利用AQS的CLH数据结构来存储CAS修改状态失败的线程，当获取锁资源的线程执行同步代码后，释放锁，会唤醒CLH中的线程，非公平锁与公平锁的不同点在源码中会讲到

**ReentrantLock构造器**
```java
//新建原子锁
private final ReentrantLock lock = new ReentrantLock();


// ReentrantLock无参构造器源码如下，所以默认为非公平锁。
    public ReentrantLock() {
        sync = new NonfairSync();
    }
// 有参构造器，公平锁创建:new ReentrantLock(true)
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

**NonfairSync$lock**

```java
//非公平加锁逻辑
final void lock() {
    // 与公平锁的不同之处如下:
    // 1. cas操作，将state值由 0 改为 1
    if (compareAndSetState(0, 1))
        // 1.1 修改state成功，则获取锁资源，并设置当前线程为独占模式，即AQS的获取锁资源的线程是本线程
        setExclusiveOwnerThread(Thread.currentThread());
    // 2. 修改失败
    else
        // 2.1 若cas操作失败，则再次获取锁资源
        acquire(1);//@See AQS$acquire
}
```

**AQS$acquire**

```java
public final void acquire(int arg) {
    // 尝试获取锁资源，若获取成功，则返回true，则不会执行与后面的操作
    if (!tryAcquire(arg) // @see Sync$nonfairTryAcquire
    	&&
        // 若失败，则addWaiter 新建节点到队列中
        // @see Sync$nonfairTryAcquire and AQS$addWaiter
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 中断当前线程
        selfInterrupt();
}
```
**Sync$nonfairTryAcquire**

>tryAcquire最终调用nonfairTryAcquire

```java
final boolean nonfairTryAcquire(int acquires) {
    // 1. 获取当前线程
    final Thread current = Thread.currentThread();
    // 2. 获取 state 状态
    int c = getState();
    // 3. 若state==0，则cas操作，若成功，则当前线程独占
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 4. 若当前线程已是独占，则可重入锁(获取锁资源的线程可多次执行同步代码块)，state自增
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 失败
    return false;
}
```
**AQS$addWaiter**

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 1. 尾节点不为空
    if (pred != null) {
        // 1.1 则新建的节点的前置指向尾节点
        node.prev = pred;
        // 1.2 cas修改新建的节点为尾节点
        if (compareAndSetTail(pred, node)) {
        	/* 添加尾节点过程:
             a. O <-p n-> O(tail) 原队列
             b. O <-p n-> O(tail) <- O(new) 新建节点，并前置指向尾节点
             c. O <-p n-> O <-p O(tail,new) cas操作将新建的节点改为尾节点
             d. O <-p n-> O <-p n-> O(tail,new) 最后新建的节点(tail) 的后置指向 原尾节点
            */
            // 设置原来的尾节点的下一节点为当前节点(现尾节点)
            pred.next = node;
            return node;
        }
    }
    // 2. 将节点插入队列，必要时进行初始化
    // @see AQS$enq
    enq(node);
    return node;
}
```
**AQS$enq**

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 1. 尾节点为空，初始化头节点，head赋值到tail
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        // 2. 若尾节点不为空
        } else {
            // 2.1 新建的节点的前置指向尾节点
            node.prev = t;
            // 2.2 修改新建节点为尾节点
            if (compareAndSetTail(t, node)) {
                // 成功后原尾节点的后置指向新节点(tail)
                t.next = node;
                return t;
            }
            // 2.3 cas操作失败自旋
        }
    }
}
```


**AQS$acquireQueued**

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前置节点
            final Node p = node.predecessor();
            // 1. 若是头节点 并且 同步状态成功
            if (p == head && tryAcquire(arg)) {
                // 1.1 成功后，设置新建节点为头节点
                setHead(node);
                // 1.2 回收原头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 2. 如果前驱节点不是头节点，或者同步状态失败了，则park当前线程并且state=-1，等待前驱节点unpark
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 2.1 park当前线程并状态为 interrupted
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 3. 若失败 取消正在进行的获取尝试，唤起<0的后置节点
        if (failed)
        	// @see AQS$cancelAcquire
            cancelAcquire(node);
    }
}
```

**AQS$cancelAcquire**

```java
private void cancelAcquire(Node node) {
        // 1. Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // 2. Skip cancelled predecessors
        Node pred = node.prev;
        // 3. 删除前置节点为cancelled的
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // 4. If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        // 5. 如果不是尾节点，或cas修改尾节点失败
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            // 5.1 前置是头节点 并且(前置的状态是SIGNAL，或者前置的状态<=0，则修改状态为SIGNAL)并且前置的线程不为空
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                // 5.1.1 后置节点不为空并且状态<=0(SIGNAL)
                if (next != null && next.waitStatus <= 0)
                    // 修改后置为前置?
                    compareAndSetNext(pred, predNext, next);
            } else {
                // 5.2 唤醒
                // @see AQS$unparkSuccessor
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

**AQS$unparkSuccessor**

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        // 1. 若状态为<0(SIGNAL)，则cas修改状态为0
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    // 2. 下一节点为空或者状态大于0
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 2.1 从尾节点遍历找到 <=0 的节点(SIGNAL)，若不为空则唤起
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 3. 后置不为空，则唤起后置的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```