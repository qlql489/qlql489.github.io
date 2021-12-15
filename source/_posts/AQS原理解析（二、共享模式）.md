---
title: AQS原理解析（二、共享模式）
date: 2018.11.18  19:30:31
tags:
    - java
    - 并发
categories: 程序人生
---

![pexels-monstera-6621468](http://image.nianlun.tech/2021/12/14/c4513f4589b937a9b2afd62e021a6b86.jpg)

上一篇介绍了AQS独占模式的原理，参考链接[AQS原理解析（一）](http://blog.nianlun.tech/2018/10/28/AQS%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89/)，这篇介绍一下AQS的共享模式如何实现的。

### 共享模式可以做什么

java concurrent包中的很多阻塞类可以一次控制多个线程的挂起和唤醒，比如`Semaphore`、`CountDownLatch`,
他们内部都继承了AQS并实现了`tryAcquireShared`,`tryReleaseShared`方法

### 共享模式逻辑

线程调用`acquireShared`方法获取锁
如果失败则创建共享类型的节点放入FIFO队列，等待唤醒
有线程释放锁后唤醒队列最前端的节点，然后唤醒所有后面的共享节点

### AQS acquireShared方法

acquireShared方法是AQS共享模式的入口

```java
    /**
     * Acquires in shared mode, ignoring interrupts.  Implemented by
     * first invoking at least once {@link #tryAcquireShared},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquireShared} until success.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquireShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     */
    public final void acquireShared(int arg) {
        //获取共享锁，小于0则放入队列，挂起线程
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

在调用`tryAcquireShared`小于零后调用`doAcquireShared`

### doAcquireShared

这个方法和独占模式的acquireQueued方法差不多，流程就是

1. 在队列尾部添加共享模式节点
2. 前一个节点如果是head并且tryAcquireShared>=0则替换当前节点为head,并唤醒后面所有共享模式节点
3. 如果前一个节点不是head，则挂起当前线程

```java
    /**
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        //和独占模式相同，在尾部添加节点，不过是设置成共享模式
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取前一个节点
                final Node p = node.predecessor();
                if (p == head) {
                    //尝试获取共享锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //和独占模式不同的地方，会唤醒后面的共享节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //挂起，具体可以看上一篇
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### setHeadAndPropagate方法

```java
    /**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        //记录当前头结点
        Node h = head; // Record old head for check below
        //把当前获取到锁的节点设置为头结点
        setHead(node);
        //propagate大于0表示后面的节点也需要唤醒
        //  h.waitStatus < 0 表示节点是可唤醒状态
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            //后继节点为空或者是共享模式则唤醒
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

### doReleaseShared 唤醒操作

```java
    /**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        
        for (;;) {
            //从头节点开始
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //是需要被唤醒的状态
                if (ws == Node.SIGNAL) {
                    //CAS方式做并发控制，设置状态为0
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;   
                    //唤醒这个节点
                    unparkSuccessor(h);
                }
                //不需要唤醒，则CAS设置状态为PROPAGATE，继续循环
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                
            }
            //头结点没有改变，则设置成功，退出循环
            if (h == head)                   
                break;
        }
    }
```

可以看出，共享模式与独占模式最大的不同就是，共享模式唤醒第一个节点后会迭代唤醒后面所有的共享节点。

只看原理可能有些抽象，以CountDownLatch为例，讲一下具体实现

### CountDownLatch

CountDownLatch的作用类似起跑线，初始时可以设置线程个数。

```java
CountDownLatch countDown = new CountDownLatch(3);
```

CountDownLatch有两个方法

* countDown 计数减一
* await 线程挂起 

使用场景：

* 比如在多线程任务中，所有任务都完成了才能继续往下执行
* 比如模拟并发场景，所有任务在一个地方等待，直到个数满足了一起执行。

#### CountDownLatch内部同步器的实现

```java
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        //初始个数
        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }
        //个数为0返回1，否则返回-1
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            //cas方式计数减一
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
    
    //初始化方法
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    
    //可中断方式获取锁，与acquireShared原理一样，额外加入了中断判断
     public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    //释放锁，计数器减一
    public void countDown() {
        sync.releaseShared(1);
    }
```

我们可以看到CountDownLatch的处理逻辑

1. 多个线程调用await方法，将共享模式节点加入到队列中，线程挂起
2. 有线程调用countDown(),尝试释放锁，计数器减一，但只有count为0时才能执行doReleaseShared方法，唤醒后面所有的共享节点，所有挂起线程一起开始执行。