---
layout: post
title: Java基本概念原理总结(三)
categories: [Java]
tags: [Summary]
---

本文是一些Java 并发编程的总结。主要来自《Java编程思想》和互联网。

### Java多线程实现方式

Java多线程实现方式主要有三种：

- 继承Thread类

- 实现Runnable接口

- 使用ExecutorService、Callable、Future实现有返回结果的多线程

其中前两种方式的多线程执行完后没有返回值，只有最后一种是带返回值的；其中最常用的也是前两种实现方式。

**继承Thread类实现多线程**

本质上来说Thread也是实现了Runnable接口的一个实例，它代表一个线程实例，并且启动线程唯一的方法就是通过Thread类的start方法；start()方法是一个native的方法，它将启动一个新线程，并执行run()方法。之中方式实现多线程很简单，通过自己的类直接extend Thread，并复写run()方法，就可以启动新线程并执行自己定义的run()方法。

**实现Runnable接口方式实现多线程**

如果自己的类已经extends另一个类，就无法直接extends Thread，此时，必须实现一个Runnable接口。[例子](http://wangqiang6028.iteye.com/blog/1887342)
事实上，当传入一个Runnable target参数给Thread后，Thread的run()方法就会调用target.run()。

**对sleep（）、wait（）、yeid（）、join（）几个方法进行下区别总结**

sleep方法与wait方法的区别：

sleep方法是静态方法，wait方法是非静态方法。
sleep方法在时间到后会自己“醒来”，但wait不能，必须由其它线程通过notify（All）方法让它“醒来”。
sleep方法通常用在不需要等待资源情况下的阻塞，像等待线程、数据库连接的情况一般用wait。

sleep/wait与yeld方法的区别：

调用sleep或wait方法后，线程即进入block状态，而调用yeld方法后，线程进入runnable状态。
 
wait与join方法的区别：

wait方法体现了线程之间的互斥关系，而join方法体现了线程之间的同步关系。
wait方法必须由其它线程来解锁，而join方法不需要，只要被等待线程执行完毕，当前线程自动变为就绪。
join方法的一个用途就是让子线程在完成业务逻辑执行之前，主线程一直等待直到所有子线程执行完毕。

### 线程同步

多线程安全问题：当多条语句在操作同一线程共享数据是，一个线程对多条语句只执行了一部分，还没有执行完， 此时另一个线程参与进来执行，导致共享数据的错误。
 
解决办法：对多条操作共享数据的语句，只能让一个线程都执行完，在执行过程中，其他线程不可以参与执行。
Java 对于多线程的安全提供了专业的解决方式。

**synchronized关键字**

线程的同步是保证多线程安全访问竞争资源的一种手段，对于同步，在具体的Java代码中需要完成一下两个操作：

*把竞争访问的资源标识为private；* 这非常重要，否则synchronized关键字就不能防止其他任务直接访问域。

*同步哪些修改变量的代码，使用synchronized关键字同步方法或代码。*

{% highlight java %}
synchronized(对象){
     代码块
     ...
}
{% endhighlight %}

**volatile关键字**

volatile修饰的变量，线程在每次使用变量的时候，都会读取变量修改后的值。volatile很容易被误用，用来进行原子性操作。

[volatile关键字修饰变量失效](http://www.cnblogs.com/aigongsi/archive/2012/04/01/2429166.html)

这个例子很好显示了 volatile这个坑及其解释。

要始终牢记使用 volatile 的限制 —— 只有在状态真正独立于程序内其他内容时才能使用 volatile —— 这条规则能够避免将这些模式扩展到不安全的用例。

[*正确使用 volatile 的模式*](http://tuoluo2004.blog.163.com/blog/static/4015241620132164153319/)很好的说明了 volatile关键字应该在什么时候正确的用。





**显式使用Lock对象**

Lock对象必须被显式创建、锁定和释放。

[同步的前提：]()

1. 必须要有两个或者两个以上的线程运行;

2. 必须是多个线程使用同一个锁;

好处：解决了多线程的安全问题;

弊端：多个线程需要判断锁，较为消耗资源;

注意： 非静态同步函数的对象锁为this，静态同步函数所使用的锁是该方法所在类的字节码文件对象，即类名.class,静态方法里的同步锁都是使用的是类的字节码对象。

用到线程的同步，随之可能会带来死锁问题。
导致死锁的原因：两个线程互相等待竞争资源，导致两边都无法得到资源，而使自己无法运行。

[生产者消费者多线程模型的例子!!!!!](http://wangqiang6028.iteye.com/blog/1887342)

**原子类**

java.util.concurrent.atomic 包中提供了以下原子类, 它们是线程安全的类, 但是它们并不是通过同步和锁来实现的, 原子变量的操作会变为平台提供的用于并发访问的硬件原语。

- AtomicBoolean -- 原子布尔

- AtomicInteger -- 原子整型

- AtomicIntegerArray -- 原子整型数组

- AtomicLong -- 原子长整型

- AtomicLongArray -- 原子长整型数组

- AtomicReference -- 原子引用

- AtomicReferenceArray -- 原子引用数组

- AtomicMarkableReference -- 原子标记引用

- AtomicStampedReference -- 原子戳记引用

- AtomicIntegerFieldUpdater -- 用来包裹对整形 volatile 域的原子操作

- AtomicLongFieldUpdater -- 用来包裹对长整型 volatile 域的原子操作

- AtomicReferenceFieldUpdater -- 用来包裹对对象 volatile 域的原子操作

引入这些类的主要原因是为了实现一种所谓的无锁定且无等待算法. 如: 比较并交换 (CAS), 它的原理是比较当前值与期望值, 如果相同则表示该变量没有发生变化。

如下面的例子使用同步和CAS方式来实现一个ID生成器。

*同步实现方式*

{% highlight java %}
class IDGenerator {
    private int id;

    public IDGenerator() {
    }

    public synchronized int nextInt() {
        return ++id;
    }
}
{% endhighlight %}

*CAS实现方式*

{% highlight java %}
class IDGenerator {
    private final AtomicInteger id = new AtomicInteger(0);

    public IDGenerator() {
    }

    public int nextInt() {
        while (true) {
            int oldID = id.get();
            int newID = oldID + 1;
            if (id.compareAndSet(oldID, newID))
                return newID;
        }
    }
}
}
{% endhighlight %}

### 线程的状态转换图

线程在一定条件下，状态会发生变化。线程变化的状态转换图如下：

![线程的状态转换图](http://stackvoid.qiniudn.com/2014-05-20-%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2%E5%9B%BE.png)

1. 新建状态（New）：新创建了一个线程对象。

2. 就绪状态（Runnable）：线程对象创建后，其他线程调用了该对象的start()方法。该状态的线程位于可运行线程池中，变得可运行，等待获取CPU的使用权。

3. 运行状态（Running）：就绪状态的线程获取了CPU，执行程序代码。

4. 阻塞状态（Blocked）：阻塞状态是线程因为某种原因放弃CPU使用权，暂时停止运行。直到线程进入就绪状态，才有机会转到运行状态。阻塞的情况分三种：
> 等待阻塞：运行的线程执行wait()方法，JVM会把该线程放入等待池中。
> 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池中。
> 其他阻塞：运行的线程执行sleep()或join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入就绪状态。

5. 死亡状态（Dead）：线程执行完了或者因异常退出了run()方法，该线程结束生命周期。

### Executor

java.util.concurrent包中的Executor用于管理Thread对象，简化并发。在Java 5之后，通过Executor来启动线程比使用Thread的start方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免this逃逸问题——如果我们在构造器中启动一个线程，因为另一个任务可能会在构造器结束之前开始执行，此时可能会访问到初始化了一半的对象用Executor在构造器中。

Executor框架包括：线程池，Executor，Executors，ExecutorService，CompletionService，Future，Callable等。

Executor接口中之定义了一个方法execute（Runnable command），该方法接收一个Runable实例，它用来执行一个任务，任务即一个实现了Runnable接口的类。ExecutorService接口继承自Executor接口，它提供了更丰富的实现多线程的方法，比如，ExecutorService提供了关闭自己的方法，以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。 可以调用ExecutorService的shutdown（）方法来平滑地关闭 ExecutorService，调用该方法后，将导致ExecutorService停止接受任何新的任务且等待已经提交的任务执行完成(已经提交的任务会分两类：一类是已经在执行的，另一类是还没有开始执行的)，当所有已经提交的任务执行完毕后将会关闭ExecutorService。因此我们一般用该接口来实现和管理多线程。

Executor接口中之定义了一个方法execute（Runnable command），该方法接收一个Runable实例，它用来执行一个任务，任务即一个实现了Runnable接口的类。ExecutorService接口继承自Executor接口，它提供了更丰富的实现多线程的方法，比如，ExecutorService提供了关闭自己的方法，以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。 可以调用ExecutorService的shutdown（）方法来平滑地关闭 ExecutorService，调用该方法后，将导致ExecutorService停止接受任何新的任务且等待已经提交的任务执行完成(已经提交的任务会分两类：一类是已经在执行的，另一类是还没有开始执行的)，当所有已经提交的任务执行完毕后将会关闭ExecutorService。因此我们一般用该接口来实现和管理多线程。

ExecutorService的生命周期包括三种状态：运行、关闭、终止。创建后便进入运行状态，当调用了shutdown（）方法时，便进入关闭状态，此时意味着ExecutorService不再接受新的任务，但它还在执行已经提交了的任务，当素有已经提交了的任务执行完后，便到达终止状态。如果不调用shutdown（）方法，ExecutorService会一直处在运行状态，不断接收新的任务，执行新的任务，服务器端一般不需要关闭它，保持一直运行即可。

Executors提供了一系列工厂方法用于创建线程池，返回的线程池都实现了ExecutorService接口。

{% highlight java %}
    public static ExecutorService newFixedThreadPool(int nThreads)
    创建固定数目线程的线程池。


    public static ExecutorService newCachedThreadPool()
    创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线   程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。
    public static ExecutorService newSingleThreadExecutor()
    创建一个单线程化的Executor。
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
    创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。
{% endhighlight%}

 [Executor参考](http://blog.csdn.net/ns_code/article/details/17465497)

 这四种方法都是用的Executors中的[ThreadFactory建立的线程]()，下面就以上四个方法做个比较.

 **newCachedThreadPool()**

- 缓存型池子，先查看池中有没有以前建立的线程，如果有，就 reuse.如果没有，就建一个新的线程加入池中。

- 缓存型池子通常用于执行一些生存期很短的异步型任务.因此在一些面向连接的daemon型SERVER中用得不多。但对于生存期短的异步任务，它是Executor的首选。

- 能reuse的线程，必须是timeout IDLE内的池中线程，缺省     timeout是60s,超过这个IDLE时长，线程实例将被终止及移出池。注意，放入CachedThreadPool的线程不必担心其结束，超过TIMEOUT不活动，其会自动被终止。

**newFixedThreadPool(int)**

-newFixedThreadPool与cacheThreadPool差不多，也是能reuse就用，但不能随时建新的线程

-其独特之处:任意时间点，最多只能有固定数目的活动线程存在，此时如果有新的线程要建立，只能放在另外的队列中等待，直到当前的线程中某个线程终止直接被移出池子

-和cacheThreadPool不同，FixedThreadPool没有IDLE机制（可能也有，但既然文档没提，肯定非常长，类似依赖上层的TCP或UDP IDLE机制之类的），所以FixedThreadPool多数针对一些很稳定很固定的正规并发线程，多用于服务器

-从方法的源代码看，cache池和fixed 池调用的是同一个底层 池，只不过参数不同:fixed池线程数固定，并且是0秒IDLE（无IDLE），cache池线程数支持0-Integer.MAX_VALUE(显然完全没考虑主机的资源承受能力），60秒IDLE  

**newScheduledThreadPool(int)**

-调度型线程池

-这个池子里的线程可以按schedule依次delay执行，或周期执行

**SingleThreadExecutor()**

-单例线程，任意时间池中只能有一个线程

-用的是和cache池和fixed池相同的底层池，但线程数目是1-1,0秒IDLE（无IDLE）

**Executor执行Callable任务**

在Java 5之后，任务分两类：一类是实现了Runnable接口的类，一类是实现了Callable接口的类。两者都可以被ExecutorService执行，但是Runnable任务没有返回值，而Callable任务有返回值。并且Callable的call()方法只能通过ExecutorService的submit(Callable<T> task) 方法来执行，并且返回一个 <T>Future<T>，是表示任务等待完成的 Future。

如果Future的返回尚未完成，则get（）方法会阻塞等待，直到Future完成返回，可以通过调用isDone（）方法判断Future是否完成了返回。

**自定义线程池**

自定义线程池，可以用ThreadPoolExecutor类创建，它有多个构造方法来创建线程池，用该类很容易实现自定义的线程池。

{% highlight java %}
public ThreadPoolExecutor (int corePoolSize, int maximumPoolSize, long         keepAliveTime, TimeUnit unit,BlockingQueue<Runnable> workQueue)
{% endhighlight %}

- corePoolSize：线程池中所保存的线程数，包括空闲线程。

- maximumPoolSize：池中允许的最大线程数。

- keepAliveTime：当线程数大于核心数时，该参数为所有的任务终止前，多余的空闲线程等待新任务的最长时间。

- unit：等待时间的单位。

- workQueue：任务执行前保存任务的队列，仅保存由execute方法提交的Runnable任务。

### 一些新的构件

**CountDownLatch**

一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。

主要方法

{%highlight java%}
 public CountDownLatch(int count);
 public void countDown();
 public void await() throws InterruptedException
 {%endhighlight%}
 
构造方法参数指定了计数的次数
countDown方法，当前线程调用此方法，则计数减一
awaint方法，调用此方法会一直阻塞当前线程，直到计时器的值为0

**CyclicBarrier**

N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。
这样应该就清楚一点了，对于CountDownLatch来说，重点是那个“一个线程”, 是它在等待， 而另外那N的线程在把“某个事情”做完之后可以继续等待，可以终止。而对于CyclicBarrier来说，重点是那N个线程，他们之间任何一个没有完成，所有的线程都必须等待。

 1. CyclicBarrier初始化时规定一个数目，然后计算调用了CyclicBarrier.await()进入等待的线程数。当线程数达到了这个数目时，所有进入等待状态的线程被唤醒并继续。 

 2. CyclicBarrier就象它名字的意思一样，可看成是个障碍， 所有的线程必须到齐后才能一起通过这个障碍。

 3. CyclicBarrier初始时还可带一个Runnable的参数， 此Runnable任务在CyclicBarrier的数目达到后，所有其它线程被唤醒前被执行。

*CountDownLatch和CyclicBarrier的区别*

> CountDownLatch 是计数器, 线程完成一个就记一个, 就像 报数一样, 只不过是递减的。
而CyclicBarrier更像一个水闸, 线程执行就想水流, 在水闸处都会堵住, 等到水满(线程到齐)了, 才开始泄流。

**DelayQueue**

DelayQueue是一个BlockingQueue，其特化的参数是Delayed。（不了解BlockingQueue的同学，先去了解BlockingQueue再看本文）
Delayed扩展了Comparable接口，比较的基准为延时的时间值，Delayed接口的实现类getDelay的返回值应为固定值（final）。DelayQueue内部是使用PriorityQueue实现的。

DelayQueue = BlockingQueue + PriorityQueue + Delayed

DelayQueue的关键元素BlockingQueue、PriorityQueue、Delayed。可以这么说，DelayQueue是一个使用优先队列（PriorityQueue）实现的BlockingQueue，优先队列的比较基准值是时间。

DelayQueue类的主要作用：是一个无界的BlockingQueue，用于放置实现了Delayed接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。注意：不能将null元素放置到这种队列中。

[DelayQueue资料一](http://www.cnblogs.com/sunzhenchao/p/3515085.html)
[DelayQueue资料二](http://www.cnblogs.com/jobs/archive/2007/04/27/730255.html)

**PriorityBlockingQueue**

PriorityBlockingQueue里面存储的对象必须是实现Comparable接口。队列通过这个接口的compare方法确定对象的priority。 

规则是：当前和其他对象比较，如果compare方法返回负数，那么在队列里面的优先级就比较搞。 

[PriorityBlockingQueue资料一](http://zzhonghe.iteye.com/blog/826757)
[PriorityBlockingQueue资料二](http://www.blogjava.net/yuyee/archive/2010/11/01/336701.html)


