---
layout: post
title: ThreadPoolExecutor应用简析
categories: [Java]
tags: [Concurrent]
---


在JDK1.5之后，直接使用Thread类并不被提倡，而Executor成为主要角色,主要原因有：

- 可复用。线程创建和销毁都需要资源，Executor中可以复用线程，降低了线程维护成本，效率提升。
- 线程数据统计。多线程执行的一下数据，线程池进行维护，方便开发。
- 资源限制和管理。看完本文你就明白了。



本文代码基于JDK1.7, 首先看一下ThreadPoolExecutor最复杂最重要的一个构造方法：

{%highlight java%}
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }

{%endhighlight%}

参数说明：

- corePoolSize 是线程池的核心线程数，通常线程池会维持这个线程数
- maximumPoolSize 是线程池所能维持的最大线程数
- keepAliveTime 和 unit 则分别是超额线程的空闲存活时间数和时间单位
- workQueue 是提交任务到线程池的入队列
- threadFactory 是线程池创建新线程的线程构造器
- handler 是当线程池不能接受提交任务的时候的处理策略

**任务处理流程**

一个新任务被submit到ThreadPoolExecutor的处理流程如下：
![ThreadPoolExecutor](http://stackvoid.qiniudn.com/2014-08-230_12917139691L3R.gif)

- 当前线程池中线程的数目小于corePoolSize，则新建线程，并处理请求（直接抄家伙运行）

- 当池子大小等于corePoolSize，把请求放入workQueue中，池子里的空闲线程就去从workQueue中取任务并处理，不添加新线程

- 当workQueue已满放不下新入的任务时，新建线程入池，并处理请求，如果池子大小撑到了maximumPoolSize就用RejectedExecutionHandler来做拒绝处理

- 当池子的线程数大于corePoolSize的时候，多余的线程会等待keepAliveTime长的时间，如果无请求可处理就自行销毁

线程池执行器也提供了提前创建初始化线程的方法：

- public boolean prestartCoreThread()
- public int prestartAllCoreThreads()

分别是预先创建一个线程和预先创建线程直到线程数到达核心线程数corePoolSize。

**Executors类中的BlockingQueue**

只是一个接口，它所表达的是当队列为空或者已满的时候，需要阻塞以等待生产者/消费者协同操作并唤醒线程。


按照Executors类中的几个工厂方法，分别使用的是：

- 无界队列：LinkedBlockingQueue。CachedThreadPool使用的是这个BlockingQueue，队列长度是无界的，适合用于提交任务相互独立无依赖的场景。有FixedThreadPool和SingleThreadPool使用。
- 直接提交：SynchronousQueue，SynchronousQueue是无界的，也就是说他存数任务的能力是没有限制的，但是由于该Queue本身的特性，在某次添加元素后必须等待其他线程取走后才能继续添加。 所以在使用SynchronousQueue通常要求maximumPoolSize是无界的。 CachedThreadPool使用的是这个BlockingQueue，通常要求线程池不设定最大的线程数，以保证提交的任务有机会执行而不被丢掉。通常这个适合任务间有依赖的场景。

也可以定制ThreadPoolExecutor时使用ArrayBlockingQueue有界队列。这个是最为复杂的使用，所以JDK不推荐使用也有些道理。与上面的相比，最大的特点便是可以防止资源耗尽的情况发生。

**RejectedExecutionHandler**

对于任务丢弃，ThreadPoolExecutor以内部类的形式实现了4个策略。分别是：

- CallerRunsPolicy：提交任务的线程自己负责执行这个任务。
- AbortPolicy：使Executor抛出异常，通过异常做处理。
- DiscardPolicy：丢弃提交的任务。
- DiscardOldestPolicy：丢弃掉队列中最早加入的任务。

在调用构造方法时，参数中未指定RejectedExecutionHandler情况下，默认采用AbortPolicy。



[近期工作繁忙，稍微空闲，将会详细分析ThreadPoolExecutor生命周期和源码]()

###More

1. [分析ThreadPoolExecutor源码](http://www.cnblogs.com/xiaoxuetu/archive/2013/05/11/3070344.html)
1. [详解ThreadPoolExecutor](http://dongxuan.iteye.com/blog/901689)
