---

title:Skywalking收集与发送链路数据部分源码解析
date: 2019-02-09 19:42:39
tags:
	- java
categories:
	- 程序人生
---

![pexels-maria-orlova-4913499](http://image.nianlun.tech/2022/02/16/d20f210c5e2006c5678d94b1dcb40d8f.jpg)

### 链路收集大体逻辑

这里先不分析skywalking是如何自动收集数据的，而是说一下agent在收集后如何存储与发送给collector，这部分的架构关系到性能开销与对服务的影响 

大体逻辑如下：

agent内部缓存维护了一个生产消费者，收集数据时将生产的数据按分区放到缓存中，消费者用多线程消费数据，将缓存的数据封装成grpc对象发送给collector




### 链路数据接收与发送
数据的接收与发送主要在类TraceSegmentServiceClient中处理
其中的一个重要属性是DataCarrier，它来实现的生产消费模式

```java 
private volatile DataCarrier<TraceSegment> carrier;
```
大致结构如下
![DataCarrier.png](http://upload-images.jianshu.io/upload_images/4702918-f02633ff4b78ebb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### DataCarrier
属性如下: 

```java
    //一个buffer的大小
    private final int bufferSize;
    //channel的大小
    private final int channelSize;
    private Channels<T> channels;
    //消费者线程池封装
    private ConsumerPool<T> consumerPool;
    private String name;
    
```
方法#produce生产数据 

```java
    public boolean produce(T data) {
        if (consumerPool != null) {
            if (!consumerPool.isRunning()) {
                return false;
            }
        }

        return this.channels.save(data);
    }
```
 channel的save方法
```java
public boolean save(T data) {
        //计算放在channel哪个位置
        int index = dataPartitioner.partition(bufferChannels.length, data);
        //重试次数
        int retryCountDown = 1;
        if (BufferStrategy.IF_POSSIBLE.equals(strategy)) {
            int maxRetryCount = dataPartitioner.maxRetryCount();
            if (maxRetryCount > 1) {
                retryCountDown = maxRetryCount;
            }
        }
        for (; retryCountDown > 0; retryCountDown--) {
            //保存成功返回true
            if (bufferChannels[index].save(data)) {
                return true;
            }
        }
        return false;
    }
```
进入到Buffer的save方法，TraceSegmentServiceClient用的策略是IF_POSSIBLE，缓存位置还有值直接返回，所以消费不过来会丢失部分数据
```java
    boolean save(T data) {
        //数组位置自增
        int i = index.getAndIncrement();
        //不为空的处理
        if (buffer[i] != null) {
            switch (strategy) {
                case BLOCKING:
                    boolean isFirstTimeBlocking = true;
                    while (buffer[i] != null) {
                        if (isFirstTimeBlocking) {
                            isFirstTimeBlocking = false;
                            for (QueueBlockingCallback<T> callback : callbacks) {
                                callback.notify(data);
                            }
                        }
                        try {
                            Thread.sleep(1L);
                        } catch (InterruptedException e) {
                        }
                    }
                    break;
                case IF_POSSIBLE:
                    return false;
                case OVERRIDE:
                default:
            }
        }
        //写入缓存
        buffer[i] = data;
        return true;
    }
```

DataCarrier的consume方法初始化消费者线程池 

```java
    public DataCarrier consume(Class<? extends IConsumer<T>> consumerClass, int num, long consumeCycle) {
        if (consumerPool != null) {
            consumerPool.close();
        }
        consumerPool = new ConsumerPool<T>(this.name, this.channels, consumerClass, num, consumeCycle);
        consumerPool.begin();
        return this;
    }

```
参数consumerClass就是TraceSegmentServiceClient类自己，实现具体的消费方法，初始化以后线程池就启动了

consumerPool方法begin 

```java
    public void begin() {
        if (running) {
            return;
        }
        try {
            lock.lock();
            //把channel分给不同的thread
            this.allocateBuffer2Thread();
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.start();
            }
            running = true;
        } finally {
            lock.unlock();
        }
    }
```

cusumerThread的run方法

```java
    @Override
    public void run() {
        running = true;

        while (running) {
            boolean hasData = consume();

            if (!hasData) {
                try {
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }

        consume();

        consumer.onExit();
    }
```
最终会调到的TraceSegmentServiceClient这个消费者的consume方法，将TraceSegment转换成grpc对象发送给collector

```java
@Override
    public void consume(List<TraceSegment> data) {
        if (CONNECTED.equals(status)) {
            final GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);
            StreamObserver<UpstreamSegment> upstreamSegmentStreamObserver = serviceStub.collect(new StreamObserver<Downstream>() {
                @Override
                public void onNext(Downstream downstream) {

                }

                @Override
                public void onError(Throwable throwable) {
                    status.finished();
                    if (logger.isErrorEnable()) {
                        logger.error(throwable, "Send UpstreamSegment to collector fail with a grpc internal exception.");
                    }
                    ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(throwable);
                }

                @Override
                public void onCompleted() {
                    status.finished();
                }
            });

            for (TraceSegment segment : data) {
                try {
                    UpstreamSegment upstreamSegment = segment.transform();
                    upstreamSegmentStreamObserver.onNext(upstreamSegment);
                } catch (Throwable t) {
                    logger.error(t, "Transform and send UpstreamSegment to collector fail.");
                }
            }
            upstreamSegmentStreamObserver.onCompleted();

            status.wait4Finish();
            segmentUplinkedCounter += data.size();
        } else {
            segmentAbandonedCounter += data.size();
        }

        printUplinkStatus();
    }
```

#### channel
channel中包含一个Buffer数组： 

```java 
    //Buffer数组
    private final Buffer<T>[] bufferChannels;
    //数据分区策略
    private IDataPartitioner<T> dataPartitioner;
    //Buffer策略
    private BufferStrategy strategy;
```
#### Buffer对象 

```java
    public class Buffer<T> {
    //对象数组
    private final Object[] buffer;
    private BufferStrategy strategy;
    //位置标记
    private AtomicRangeInteger index;
    private List<QueueBlockingCallback<T>> callbacks;
    ... 
    
```

### 抽样服务SamplingService
作用是对TraceSegment进行抽样，链路跟踪必须要考虑的功能，服务压力大时全量收集会占用cpu、内存、网络等资源 

agent通过`agent.config`配置档中的`agent.sample_n_per_3_secs`设置每三秒收集的TraceSegment的个数，大于0为开启状态，默认全量收集

初始化一个3秒的定时任务

```java
    @Override
    public void boot() throws Throwable {
        if (scheduledFuture != null) {
            scheduledFuture.cancel(true);
        }
        //大于0开启抽样
        if (Config.Agent.SAMPLE_N_PER_3_SECS > 0) {
            on = true;
            this.resetSamplingFactor();
            ScheduledExecutorService service = Executors
                .newSingleThreadScheduledExecutor(new DefaultNamedThreadFactory("SamplingService"));
            //定时任务
            scheduledFuture = service.scheduleAtFixedRate(new RunnableWithExceptionProtection(new Runnable() {
                @Override
                public void run() {
                    resetSamplingFactor();
                }
            }, new RunnableWithExceptionProtection.CallbackWhenException() {
                @Override public void handle(Throwable t) {
                    logger.error("unexpected exception.", t);
                }
            }), 0, 3, TimeUnit.SECONDS);
            logger.debug("Agent sampling mechanism started. Sample {} traces in 3 seconds.", Config.Agent.SAMPLE_N_PER_3_SECS);
        }
    }
```
抽样逻辑

```java
    public boolean trySampling() {
        if (on) {
            int factor = samplingFactorHolder.get();
            if (factor < Config.Agent.SAMPLE_N_PER_3_SECS) {
                boolean success = samplingFactorHolder.compareAndSet(factor, factor + 1);
                return success;
            } else {
                return false;
            }
        }
        return true;
    }
```
skywalking的agent不支持按比例抽样，比较遗憾