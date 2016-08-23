---
layout: post
title: Java对象的生命周期
categories: [Java]
tags: [JVM]
---

Java对象被创建前，此Java类字节码（后缀.class的文件）必须从文件系统加载（包括类加载、准备和初始化）到JVM。所有的Java对象都是在堆上被创建出来。

在JVM运行空间中，将对象生命周期分为七个阶段：

1. 创建阶段
1. 应用阶段
1. 不可视阶段
1. 不可到达阶段
1. 可收集阶段
1. 终结阶段
1. 释放阶段

###创建阶段

在对象的创建阶段，系统主要通过下面的步骤，完成对象的创建过程： 

- 为对象分配存储空间； 
- 开始构造对象； 
- 第一次创建对象从超类到子类对static成员进行初始化（类的static成员的初始化是在ClassLoader第一次加载该类的时候）； 
- 超类成员变量按顺序初始化，递归调用超类的构造方法； 
- 子类成员变量按顺序初始化，子类构造方法调用。 

>在创建对象时应注意几个关键应用规则： 
        
> 尽量及时使对象符合垃圾回收标准。比如 myObject = null（如果要提高软件性能，一定不要忽视这一点！这点非常重要，其实JVM没你想象的智能）。 
> 不要采用过深的继承层次。 
> 访问本地变量优于访问类中的变量。 

###应用阶段

在对象的引用阶段，对象具备如下特征： 

- 系统至少维护着对象的一个强引用(Strong Reference); 
- 所有对该对象的引用全部是强引用(除非我们显示地适用了：软引用(Soft Reference)、弱引用(Weak Reference)或虚引用(Phantom Reference)). 

> 强引用(Strong Reference)：是指JVM内存管理器从根引用集合出发遍历堆中所有到达对象的路径。当到达某对象的任意路径都不含有引用对象时，这个对象的引用就被称为强引用。 

> 软引用(Soft Reference)：软引用的主要特点是有较强的引用功能。只有当内存不够的时候，才回收这类内存，因此内存足够时它们通常不被回收。另外这些引用对象还能保证在Java抛出OutOfMemory异常之前，被设置为null。它可以用于实现一些常用资源的缓存，实现Cache功能，保证最大限度地使用内存你而不引起OutOfMemory。 

> 弱引用(Weak Reference)：弱应用对象与软引用对象的最大不同就在于：GC在进行垃圾回收时，需要通过算法检查是否回收Soft应用对象，而对于Weak引用，GC总是进行回收。Weak引用对象更容易、更快地被GC回收。Weak引用对象常常用于Map结构中。

> 虚引用(Phantom Reference)对象指一些执行完了finalize函数，并为不可达对象，但是还没有被GC回收的对象。这种对象可以辅助finalize进行一些后期的回收工作(最好不要使用finalize)。

在实际程序设计中一般很少使用弱引用和虚引用，是用软引用的情况较多，因为软引用可以加速JVM对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出(OutOfMemory)等问题的产生。 

{%highlight java%}

 try {
             Object localObj = new Object();
	       localObj.doSomething();
       } catch (Exception e) {
           e.printStackTrace();
       }

       if (true) {
	    // 此区域中localObj 对象已经不可视了, 编译器会报错。
	    localObj.doSomething();
       }
{%endhighlight%}

###不可到达阶段 

处于不可达阶段的对象在虚拟机的对象引用根集合中再也找不到直接或间接地强引用，这些对象一般是所有线程栈中的临时变量。所有已经装载的静态变量或者是对本地代码接口的引用。 

###可收集阶段、终结阶段与释放阶段 

当一个对象处于可收集阶段、终结阶段与释放阶段时，该对象有如下三种情况： 

- 回收器发现该对象已经不可达。 

- finalize方法已经被执行（finalize方法已经被抛弃，完全不赞成在项目里使用这个方法，会造成很多不确定性）。 



###参考

1. [Object Lifecycle](http://en.wikibooks.org/wiki/Java_Programming/Object_Lifecycle)

2. [Object lifetime](http://en.wikipedia.org/wiki/Object_lifetime#Java)

