---
title: AQS原理解析（一）
date: 2018-10-28 12:37:34
tags:
    - java
    - 并发
categories: 程序人生
---

### AQS是什么
java concurrent包中有很多阻塞类如：`ReentrantLock`、`ReentrantReadWriteLock`、`CountDownLatch`、`Semaphore`、`Synchronous`、`FutureTask`等，他们的底层都是根据aqs构建的，它可以说是java多线程编程最底层核心的抽象类。既然这么重要，我们就来看看它底层原理到底是什么。 

aqs全称`AbstractQueuedSynchronizer`，它作为抽象类无法单独使用，需要有具体实现，不同的实现中自己定义什么状态意味着获取或者被释放

### AQS的原理是什么
AQS内部维护一个先进先出（FIFO）的等待队列叫做CLH队列，当一个线程来请求资源时，AQS通过状态判断是否能获取资源，如果不能获取，则挂起这个线程，和状态一起封装成一个Node节点放在队尾，等待前面的线程释放资源好唤醒自己，所以谁先请求的谁最先获得机会唤醒,当然新线程可能加塞提前获取资源，在源码解析可以看到原因

![aqs.jpg](http://image.nianlun.tech/2018/10/28/64a59fef2d4a47dae1902a65d32c0701.jpg)


AQS分独占和共享两种方式，独占模式，只有一个线程可以获得锁，比如ReentrantLock，共享模式下可以允许多个线程同时获取锁，比如CountDownLatch使用的就是共享方式，
### 源码解析

AQS的子类需要实现的方法 

```java
    //独占方式获取资源
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    
    //独占释放资源
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
    //共享获取资源
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }
    
    //共享释放资源
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
    
    //是否独占
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
```
可以看到，子类调用这些方法如果没有实现的话会抛异常，当然也不是所有方法都要实现，找自己需要的实现就可以了。

为了更好的理解先实现一个最简单的锁,只需要实现`tryAcquire`和`tryRelease`方法即可

```java
public class TestLock {

    private Sync sync = new Sync();
    //加锁
    public void lock(){
        sync.acquire(1);
    }
    //解锁
    public void unLock(){
        sync.release(1);
    }


    public static class Sync extends AbstractQueuedSynchronizer {

        @Override
        protected boolean tryAcquire(int arg) {
            assert arg == 1;
            //cas将状态从0设为1，如何不为0则失败
            if(compareAndSetState(0,1)){
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            assert arg == 1;
            if(getState() == 0){
                throw new IllegalMonitorStateException();
            }
            //将状态设为0
            setState(0);
            return true;
        }

    }
}
```
再来写一个并发场景，简单的加法，先获取前值，用sleep模拟方法执行时间比较长,然后累加

```java
public static void main(String[] args) {

        final AddCount count = new AddCount();

        ExecutorService executorService = Executors.newCachedThreadPool();
        for(int i = 0;i<3;i++){
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        count.add(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    public static class AddCount{

        private int countTotle = 0;

        public void add(int count) throws InterruptedException {

            int tmp = this.countTotle;

            Thread.sleep(100L);

            this.countTotle = tmp+count;
            System.out.println(this.countTotle);
        }

    }
    //输出
    100
    100
    100
```
在add方法加上自定义的的锁

```java
public static void main(String[] args) {

        final AddCount count = new AddCount();
        final TestLock testLock = new TestLock();

        ExecutorService executorService = Executors.newCachedThreadPool();
        for(int i = 0;i<3;i++){
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        testLock.lock();
                        count.add(100);
                        testLock.unLock();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }
    //输出
    100
    200
    300
```

根据这个简单的例子，我们来看一下源码中是怎么实现的

#### acquire
lock方法首先调用的是AQS的`acquire`方法

```java
	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

它会调用`tryAcquire`尝试去取锁，如果没有取到的话调用`addWaiter`将Node放入队尾，同样也使用CAS的方式，AQS中有大量CAS的使用，不了解CAS的可以看[浅析乐观锁、悲观锁与CAS](https://www.jianshu.com/p/385741045050) 

这里有新的线程在执行第一个判断`!tryAcquire(arg)`时，如果刚好有线程释放锁，那新的线程很有可能插队直接获取到锁，也就是有队列也无法公平的原因。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

在尾部添加node，将node双向关联，如果成功则直接返回，这里有一个问题，在设置队尾的时候，没有并发控制，有另一个线程也来设置，就只会有一个线程成功，没成功的线程或者队尾为空则执行enq方法。

##### enq方法 

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

这里看到如果tail是null,则cas设置head为一个新节点,也就是说第一个入队的节点head和tail是相同的。
如果队尾不为空，则用cas加自旋的方式放入队尾。

##### Node对象
node对象封装了状态和请求的线程以及前后节点的地址

```java
static final class Node {
        //共享节点
        static final Node SHARED = new Node();
        //非共享节点
        static final Node EXCLUSIVE = null;

        //取消状态（因超时或中断）
        static final int CANCELLED =  1;
        //等待唤醒
        static final int SIGNAL    = -1;
        //等待条件
        static final int CONDITION = -2;
        //对应共享类型释放资源时，传播唤醒线程状态
        static final int PROPAGATE = -3;
        //当前状态
        volatile int waitStatus;
        //前一个节点
        volatile Node prev;
        //下一个节点
        volatile Node next;
        //请求的线程
        volatile Thread thread;

        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        //获取前一个节点，为空则抛空指针异常
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }

    }
```

没有使用condition，node常用的状态有 0 新建状态和 -1 挂起状态

##### acquireQueued
再看一下acquireQueued方法

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取前一个节点
                final Node p = node.predecessor();
                //如果前一个节点是head,并且能获取锁，则将当前节点设置为head
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //判断前面节点的状态，中断当前线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                //失败了设置成取消状态
                cancelAcquire(node);
        }
    }
    
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //如果前一个节点已经是等待状态，可以安全park
        if (ws == Node.SIGNAL)
            return true;
        //如何前一个节点是取消状态了，则一直往前取，去掉取消状态的节点，直到状态不为取消状态的节点
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //ws必须是0或-3才会走这里，cas设置成-1待唤醒状态
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    
    //中断当前线程
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

这里的主要逻辑就是将新加入的节点设置为待唤醒状态，进入队列的节点都进入中断状态，head节点持有锁，锁被释放后后面的节点会代替之前的head成为新的head节点
##### release
释放锁的过程，掉用`release`方法
    
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    private void unparkSuccessor(Node node) {

        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        //清除取消状态的节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒后一个等待的线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

调用LockSupport.unpark后，唤醒后一个中断的线程，队列剔除之前的head，这样往复，释放锁后继续唤醒后面的线程。



