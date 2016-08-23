---
layout: post
title: Android 中的线程调度
categories: [Android]
tags: [Tips]
---
<Br />
本文概述了 Android 中的线程是如何调度的，并通过设置线程优先级来优化 APP-UI，使其流畅运行。

线程调度听起来很学术，是操作系统中的概念：线程调度决定系统中不同线程运行，运行时间，何时运行。Android 中的线程调度很操作系统中的线程调度类似，主要使用 nice 和 cgroups 这两个变量来调度线程(本质上来说还是通过设置线程优先级，让 Linux 内核有依据的分配线程运行)。
<Br />
### <font color="green">Nice</font>
-----------------------------------

我们这里的 Nice 与 Linux Kernel 的进程完全公平调度器(CFS)类似但是不一样(本质上，在创建线程的时候这个 Nice 值最终还是要传递给底层的，跟 Linux Kernel 中的 nice 是相同的)。Java 层

**Nice** 在 Android 中是衡量线程优先级的重要参考指标；**nice** 值越大，线程优先级越低，反之越高；其中两个重要的线程优先级属性参数是，<font color="green">default 和 background </font>， 对于 Android App，线程要做大量耗时工作，其优先级应该越低，否则系统有卡死的风险，反之，线程做的工作不多，应该设置其优先级较高。所以关于交互界面的线程(如 UI 线程等前台线程)会给 <font color="green">default</font> 或更高的优先级，而后台线程(如执行 AsyncTask 的线程)的线程属性为<font color="green">background</font>。

**Nice** 这种特性可以让后台工作线程尽量少抢占前台线程的时间片(从OS来说)，从而保证了用户界面的流畅性。实际上，仍有一种可能使 UI 卡死，例如有30个后台线程，但是只有1个 UI 进程，这30个后台线程可能占据了很大的计算资源，导致 UI 线程得不到及时运行，导致卡顿、掉帧；为了防止这种情况发生， Android 提出了 **cgroups**，用来解决类似的问题。
<Br />
### <font color="green">Cgroups</font>
----------------------------------------

为了解决上述问题，Android 将前台线程和后台线程分开运行，即利用Linux的 *cgroups* 给前台线程建一个群，给后台线程建一个群，然后调度器在这两个群中跑，这样不管后台线程有多少，调度器总能照顾到前台线程的运行；所以用户体验得到了保证。

*如何区分前台进程和后台进程的 cgroup？*

Android 会自动将没有在前台进程 cgroup 运行的线程都放到后台进程的 cgroup 中，这就保证了不管开多少线程，只会将其分为两个 Group，前台 cgroup 和后台 cgroup；理论上，不管多少非前台线程运行，Android都能保证前台线程的运行。
<Br />
### <font color="red">设置线程属性</font>
----------------------------
一般来说，Android API 将常用工作线程的优先级设置好了。例如，[HandlerThread](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/os/HandlerThread.java) 的代码第30行，将线程优先级设置为**Process.THREAD_PRIORITY_DEFAULT**；[AsyncTask](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/os/AsyncTask.java) 代码第286行，将线程优先级设置为**Process.THREAD_PRIORITY_BACKGROUND**。再比如音乐播放的线程设置为**THREAD_PRIORITY_AUDIO**，Android API 将[常用情况都做了详细的分类](http://developer.android.com/reference/android/os/Process.html)。

需要铭记于心的是，在 UI 主线程实例化的线程或线程池，都会继承 *default* 或 *foreground*属性(因为 UI 线程就是这个属性，子线程会继承父线程的属性)，所以如果大量类似工作线程存在于前台进程的 cgroup 中话，可能会使 UI 变得非常不流畅。若是工作线程，建议在运行前通过 *Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)* 直接设置线程的属性。

{%highlight java linenos%}

new Thread(new Runnable() {
  @Override
  public void run() {
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

    // ...
  }
}).start();

{%endhighlight%}
