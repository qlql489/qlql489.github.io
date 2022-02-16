---
title: AQS原理解析（三、condition原理）
date: 2020-03-01 13:44:19
tags:
    - java
    - 并发
categories: 程序人生
---

![pexels-markus-winkler-3828944](http://image.nianlun.tech/2022/02/16/3268df139bfcd085d1c25b09df4beef6.jpg)

### condition的作用

condition的使用场景其实很多，涉及到条件判断的并发场景都可以用到，比如： 

* 阻塞队列的ArrayBlockingQueue中做队列满和空的条件判断
* CyclicBarrier中做阻塞与唤醒所有线程的判断
* DelayQueue中的阻塞获取队列数据的判断
* 线程池ThreadPoolExecutor中awaitTermination方法的条件判断

condition怎么用呢？

在使用synchronized时我们可以使用wait()、notify()、notifyAll()方法来调度线程，而condition提供了类似的方法：wait(),signal(),signalAll的功能，并且能够更加精细的控制等待的范围，像上面所说，jdk中使用了很多ReentrantLock和condition的配合来实现线程调度

我们看一个conditon最常见的使用方式：生产消费者的模型：


```
public class ConditionTest {

    LinkedList<String> lists = new LinkedList<>();

    Lock lock = new ReentrantLock();

    //集合是否满的条件判断
    Condition fullCondition = lock.newCondition();

    //集合是否空的条件判断
    Condition emptyCondition = lock.newCondition();

    //生产者
    private void product(){
        lock.lock();
        try {
            //假如集合大小为10
            while (lists.size() == 10){
                System.out.println("list is full");
                fullCondition.await();
            }
            //生产一个5位的随机字符串
            String randomString = getRandomString(5);
            lists.add(randomString);
            System.out.println(String.format("product %s size %d  %s",randomString,lists.size(),Thread.currentThread().getName()));
            //通知消费者可以消费了
            emptyCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    //消费者
    private String consume(){
        lock.lock();
        try{
            while (lists.size() == 0){
                System.out.println("list is empty");
                emptyCondition.await();
            }
            String first = lists.removeFirst();
            //通知生产者可以生产了
            fullCondition.signalAll();
            return first;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
        return null;
    }
    
    /**
     * 生成随机字符串
     * @param length
     * @return
     */
    public static String getRandomString(int length){
        String str="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        Random random=new Random();
        StringBuffer sb=new StringBuffer();
        for(int i=0;i<length;i++){
            int number=random.nextInt(62);
            sb.append(str.charAt(number));
        }
        return sb.toString();
    }

    public static void main(String[] args) {

        ConditionTest test = new ConditionTest();

        ExecutorService executorService = Executors.newCachedThreadPool();

        //线程个数控制消费的快还是生产的快
        for(int i = 0;i<2;i++){

            executorService.submit(()->{
                System.out.println(Thread.currentThread().getName());
                while (true){
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.product();
                }
            });
        }

        for(int k = 0;k<1;k++){
            executorService.submit(()->{
                System.out.println("cousumestart");
                while (true) {
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    String consume = test.consume();
                    System.out.println("consume " + consume+ " "+Thread.currentThread().getName() );
                }
            });
        }

        //等待输入，阻塞主线程不退出
        try {
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        } catch (IOException e) {
            e.printStackTrace();
        }


    }
```

```
//部分输出日志
product qeV0r size 7  pool-1-thread-1
product xEUkA size 8  pool-1-thread-2
consume P5Je1 pool-1-thread-3
product rQS1D size 8  pool-1-thread-1
product QcEtf size 9  pool-1-thread-2
consume 2q7Fc pool-1-thread-3
product Z5rBg size 9  pool-1-thread-1
consume UBxBD pool-1-thread-3
product Tr5q2 size 9  pool-1-thread-2
product HXBdE size 10  pool-1-thread-1
list is full
consume aYDNR pool-1-thread-3
product ukjnk size 10  pool-1-thread-2
list is full
consume LBEdA pool-1-thread-3
product iK28H size 10  pool-1-thread-2
list is full
list is full
```

可以看到生产者线程有2个，消费者线程有1个，生产和消费的速度相同，用Thread.sleep控制，
生产速度大于消费速度，最后集合元素到10个的时候生产者调用`fullCondition.await();`阻塞，只有消费者消费后通过`fullCondition.signalAll();`通知生产者继续生产

同理添加消费者线程数，使消费的速度快与生产，则集合为空时会调用`emptyCondition.await();`阻塞，生产者生产后回调用`emptyCondition.signalAll();`通知消费者继续生产

相较于对象的wait()、notifyAll()方法不同的条件分开判断，颗粒度更小一些，唤醒的线程范围更精准


再看一下ArrayBlockingQueue的一个例子，在一段时间内阻塞获取队列数据，取不到则返回空：

```
  public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                //notEmpty 是lock new出来的一个condition
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

condition的使用场景还多，下面我们就一起看看condition的实现原理吧，首先condition需要在AbstractQueuedSynchronizer实现类的

### condition原理解析

我们知道AQS中维护了一个队列来控制线程的执行，condition中使用了另一个等待队列来实现条件的判断，condition必须在aqs的acquire获取锁后使用，调用condition.await()方法将添加一个node到条件队列中，在调用signal()或signalAll()后将此节点移出condition的等待队列放到锁的等待队列中去竞争锁，取到锁后继续执行后续逻辑。

---



condition有以下几个方法

```java
//将等待时间最长的线程从condition等待队列放到锁的等待队列中
public final void signal()
//将所有等待线程从condition等待队列放到锁的等待队列中
public final void signalAll()
//condition的等待方法
public final void await() throws InterruptedException 
//不可中断的wait
public final void awaitUninterruptibly()
//几个有时间参数的wait方法
public final long awaitNanos(long nanosTimeout)
                throws InterruptedException
public final boolean awaitUntil(Date deadline)
                throws InterruptedException
public final boolean await(long time, TimeUnit unit)
                throws InterruptedException               
```

#### 先看一下最主要的await方法

AbstractQueuedSynchronizer.ConditionObject#await()

```java

        public final void await() throws InterruptedException {
          	//如果当前线程被中断了抛出InterruptedException
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();//（1）
            int savedState = fullyRelease(node);//(2)
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {//(3)
              	//挂起线程
                LockSupport.park(this);
              	//中断情况的判断
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //被唤醒后去抢锁，抢到后继续执行
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
             //如果阻塞中发生了中断，则抛出异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        
```



##### （1）addConditionWaiter

在condition等待队列尾部加入一个节点

```java
			private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 如果最后一个节点不是condition状态（被取消状态）被取消状态是在fullyReleas方法中产生的
            if (t != null && t.waitStatus != Node.CONDITION) {
                //从头节点开始将被取消或者超时的节点移出队列
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
          	//队列为空的情况  
        		if (t == null)
                firstWaiter = node;
            else
              	//插入尾节点
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

##### （2）fullyRelease

能调用wait方法说明已经获取到锁了，fullyRelease方法就是提前调用解锁方法，将自己从lock的队列中移出，并返回当前节点的状态savedState，这里如果释放失败说明当前线程不在持有锁，状态错误，将节点设置成CANCELLED状态

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

release方法调用tryRelease释放锁并唤醒首节点，在ReentrantLock的实现中tryRelease会判断当前线程是否获取锁，所以在lock方法范围内使用condition会报IllegalMonitorStateException异常

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
```

##### （3）isOnSyncQueue

回到await方法，循环调用isOnSyncQueue判断是否在锁的等待队列中(注意不是condition的等待队列)，不在锁的等待队列中则调用`LockSupport.park(this)`挂起线程。

```java
		final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        
        return findNodeFromTail(node);
    }
```

#### awaitNanos方法

大致逻辑和await相同，就是多了一个时间的判断

```java
        
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
            		//如果时间小于0，直接从condition队列
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                //如果大于自旋的阈值则使用parkNanos设置线程挂起的时间，否则继续自旋
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }
```
#### signal()方法

signal的作用是将condition队列中等待时间最长的node转移到锁队列末尾，去重新抢锁
```java

        public final void signal() {
        		//有不同的实现，ReentrantLock中是判断持有锁的是否当前线程
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

##### doSignal
将condition中等待时间最长的节点调用transferForSignal方法放到锁队列中，循环调用是要寻找第一个不是cancelled状态的节点

```
       
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```
##### doSignalAll
doSignalAll是将所有等待队列中的节点放到锁队列末尾
```
       
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```

##### transferForSignal

```
    final boolean transferForSignal(Node node) {
      
        //cas设置节点为0状态，如果失败说明节点已经被取消了
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        //添加到锁队列中
        Node p = enq(node);
        int ws = p.waitStatus;
        //cancelled状态或者设置SIGNAL状态失败则唤醒此线程
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
condition中有很多线程与中断的细节处理，有兴趣的可以自己去看看源码

总结一下：
* condition必须使用在lock中
* condition提供了类似object.wait和notify的通信机制，但支持多个条件队列，使用上更灵活
* condition的原理流程如下
  * 线程1获取锁
  * 线程1调用condition.await()进入condition等待队列并阻塞，释放锁给别的线程
  * 线程2获取锁，调用condition.signal，将condition等待队列中的线程1所在的node放在锁的等待队列中竞争锁