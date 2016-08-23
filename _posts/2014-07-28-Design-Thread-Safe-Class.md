---
layout: post
title: 如何实现线程安全类
categories: [Java]
tags: [Concurrent]
---


假设我们有一个简单的类 MyClass,下面我将用如下几种方法来实现这个类线程安全。

{%highlight java%}
public class MyClass {
   private int number;
   public MyClass(int number) {
        this.number = number;
    }
    public int getNumber() {
        return number;
    }
    public void setNumber(int number) {
        this.number = number;
    }
}
{%endhighlight%}

**使用final关键字**

final关键字使变量一旦初始化，就不会改变，相当于生成了一个不可变类。String类 Integer类都是不可变类。

{%highlight java%}

public class MyClass {
   private final int number;

   public MyClass(int number) {
    this.number = number;
   }

   public int getNumber() {
    return number;
   }

}

{%endhighlight%}

**使用关键字volatile**

volatile关键字使变量对线程可见。要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件：

- 对变量的写操作不依赖于当前值。

- 该变量没有包含在具有其他变量的不变式中。

{%highlight java%}

public class MyClass {
   private volatile int number;

   public MyClass(int number) {
    this.number = number;
   }

   public int getNumber() {
       return number;
   }

   public void setNumber(int number) {
       this.number = number;
   }
}

{%endhighlight%}

**使用synchronized关键字**

Java中最基本的同步手段就是synchronized关键字，synchronized关键字经过编译后，会在同步块前后分别形成monitorenter(执行之前要先尝试获得此锁)和monitorexit这两个字节码指令，这两个字节码都需要一个reference类型的参数来致命要锁定和解锁的对象。
如果Java程序中的synchronized明确指定了对象参数，那就是这个对象的reference；若没有明确指定，那就根据synchronized修饰的
是实例还是类方法，去取对应对象实例或Class对象来作为锁对象。

**注意：**synchronized同步块对同一线程来说是可重入的，不会出现自己吧自己锁死的情况，其次同步块在已进入的线程执行前，会阻塞后面其他线程的进入。Java线程是映射到操作系统原生线程上的，阻塞或唤醒线程，需要操作系统从用户态转到核心态，会消耗较多时间，所以synchronized是Java语言中一个重量级(Heavyweight)操作，在有必要的情况下才使用。

{%highlight java%}

public class MyClass {
   private int number;

   public MyClass(int number) {
       setNumber(number);
   }

   public synchronized int getNumber() {
       return number;
   }

   public synchronized void setNumber(int number) {
       this.number = number;
   }
}

{%endhighlight%}

**使用ReentrantLock**

相比synchronized关键字，ReentrantLock增加了一些高级功能，主要有：等待可中断，可实现公平锁以及锁可绑定多个条件。

- 等待可中断：指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块有很大帮助。
- 公平锁：指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；非公平锁则不保证，锁被释放，任何一个线程都可能获得。synchronized中的锁时非公平的，ReentrantLock默认也是非公平锁，但可以通过布尔值的构造函数来要求使用公平锁。
- 锁绑定多个条件是指一个ReentrantLock对象可以同时绑定多个Condition对象。

一个例子。
首先需要定义一个lock，在使用时首先通过lock的lock方法加锁，然后执行临界区代码，最后在final中调用lock的unlock方法解锁（防止异常后无法解锁）

{%highlight java%}
public class ReentrantLockTest {
	private final ReentrantLock lock = new ReentrantLock();//lock最好定义为final

	public void m() {
		lock.lock();
		try {
			// method body
		} finally {
			lock.unlock();
		}
	}
}
{%endhighlight%}


**无需同步方案**

若一个方法不涉及共享数据，那就无需同步，天生线程安全。下面介绍两类。

- 可重入代码(Reentrant Code)：可在代码执行的任何时刻打断它，转而去执行另一段代码（或递归调用本身），控制权返回后，原来的程序不会出现任何错误；可重入性就是线程安全的，反之则不成立。
- 线程本地存储（Thread Local Storage）：

ThreadLocal是JDK的一个线程本地存储的类，我们可以把一些线程私有的数据写在ThreadLocal中，这样这些数据只有一个线程可见，实现了所谓的栈封闭。这样存储一些线程私有的数据，我们就不用去费心考虑如何保证临界资源的互斥访问了，同时对于一个线程，这些私有数据也只做一次初始化。 

向ThreadLocal里面存东西就是向它里面的Map存东西的，然后ThreadLocal把这个Map挂到当前的线程底下，这样Map就只属于这个线程了。一个线程可以有多个ThreadLocal实例，那么有多少个ThreadLocal实例就决定了Map的大小，这个数组是动态增长的，每次要是大小不够，就自动扩充为原来大小的2倍，然后对于原来的元素重新Hash,这个操作的成本还是很大的。 

[ThreadLocal的用法和分析](http://suo.iteye.com/blog/1342393)

**参考**

深入理解Java虚拟机
