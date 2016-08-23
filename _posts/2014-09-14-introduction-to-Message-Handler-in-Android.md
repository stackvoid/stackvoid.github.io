---
layout: post
title: 理解Android应用的消息机制
categories: [Android]
tags: [Handler]
---


本文代码分析均基于Android4.4。

Android应用程序主线程用来跟新UI，所以不能让主线程做费时操作，否则会出现ANR(App Not Response), 一般来说耗时操作都新开启一个线程，新线程执行结束，发消息给主线程来更新UI，常用方法有：

- Activity.runOnUiThread(Runnable)
- View.post(Runnable)
- View.postDelayed(Runnable, long)
- Hanlder机制

**Activity.runOnUiThread源码**

方法runOnUiThread()位于base/core/java/android/app/Activity.java中，发现此方法首先判断当前线程是不是主线程(UI线程)，不是的话使用Handler(post到消息队列中，后面会详解)，是的话调用run方法。

{%highlight java%}
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action);
    } else {
        action.run();
    }
}
{%endhighlight%}

**View.post源码**

抛开AttachInfo等这些与本文无关的信息，发现View.post最终还是使用handler来传送Runnable对象。同样View.postDelayed方法也同样使用Handler。

{%highlight java%}
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
        //postDelayed方法使用下面的方法
        //return attachInfo.mHandler.postDelayed(action, delayMillis);
    }
    // Assume that post will succeed later
    ViewRootImpl.getRunQueue().post(action);
    //postDelayed方法使用下面的方法
    //ViewRootImpl.getRunQueue().postDelayed(action, delayMillis);
    return true;
}
{%endhighlight%}

分析到这里，我们发现Android中线程到主线程之间的消息传递虽然有这四种方法，其本质上都是使用了Handler。下面我们将揭开Handler的面纱。

如下图，Android使用消息机制(Handler-Looper机制)实现线程的通信，线程通过Looper建立自己的消息循环，MessageQueue是FIFO的消息队列，Looper负责从MessageQueue中取出消息，并且分发到消息指定目标Handler对象。Handler对象绑定到线程的局部变量Looper，封装了发送消息和处理消息的接口。

![handler01](http://stackvoid.qiniudn.com/2014-09-14-handler01.png)


**Handler类构造方法**

{%highlight java%}
    public Handler(Callback callback, boolean async) {

    	.........

        mLooper = Looper.myLooper();
        //如果是普通线程，想成为Looper的线程，必须先调用Looper.prepare()
        //若是主线程(UI线程)，在Activity启动时，已经调用过Looper.prepare()
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //不管new 多少个Handler，这些Handler都共用Looper里的消息队列，
        //由于一个线程只能有一个ThreadLocal<Looper>对象,则消息队列只有一个
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    //myLooper方法返回Looper对象，Looper对象时线程本地存储(ThreadLocal)
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

    //下面这个方法是Looper.java中的方法，prepare方法的作用就是创建线程中
    //唯一的Looper对象，若已经有Looper对象，再次调用prepare方法则抛出异常
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
{%endhighlight%}

总结一下，Handler构造方法通过获取当前线程唯一的Looper对象来初始化消息队列(其实是共享Looper的mQueue，为什么这么做，读完本文你就明白了)。


**应用程序启动自动加载Looper**

Android应用程序进程在启动的时候，会在进程中加载ActivityThread类，并且执行这个类的main函数，应用程序的消息循环过程就是在这个main函数里面实现的，定义在frameworks/base/core/java/android/app/ActivityThread.java文件中

{%highlight java%}
public final class ActivityThread {
	......

	public static final void main(String[] args) {
		......
		//将当前线程初始化为Looper线程。最终会调用Looper.prepare()
		Looper.prepareMainLooper();

		......
 	// 开始循环处理消息队列
		Looper.loop();

		......
	}
}
{%endhighlight%}

主线程中可以直接创建Handler对象，而在子线程中需要先调用Looper.prepare()才能创建Handler对象。

由前面Handler构造方法我们知道，应用程序的主线程中会始终存在一个Looper对象。

所以我们继承Activity实现我们自定义的Activity中，直接使用Handler就好了(Looper已经建立好了)。

[**消息发送和接收**}()

消息发送和接收流程相信大家也已经非常熟悉了，new出一个Message对象，然后使用setData()方法或arg参数等方式为消息携带一些数据，再借助Handler将消息发送出去就可以了，发出去后，主线程中创建Handler对象时实现其handleMessage方法即可。

{%highlight java%}
new Thread(new Runnable() {
	@Override
	public void run() {
		Message message = new Message();
		message.arg1 = 1;
		Bundle bundle = new Bundle();
		bundle.putString("data", "data");
		message.setData(bundle);
		handler.sendMessage(message);
	}
}).start();

Handler handler = new Handler(){
	@Override
	public void handleMessage(Message msg){
		..........//处理Message的实现
	}
}
{%endhighlight%}

我们现在来看一下发送到接收到这个数据的全过程，即从handler.sendMessage(message)到handleMessage(Message msg)的全过程。Handler中提供了很多个发送消息的方法，其中除了sendMessageAtFrontOfQueue()方法之外，其它的发送消息方法最终都会调用到sendMessageAtTime()方法中，方法的源码如下：

{%highlight java%}
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    //这个enqueueMessage是Handler中的方法，仅仅是一个封装而已
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;//方便出队列的时候找到自己的Handler
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }    
{%endhighlight%}

sendMessageAtTime()方法接收两个参数，其中msg参数就是我们发送的Message对象，而uptimeMillis参数则表示发送消息的时间，它的值等于自系统开机到当前时间的毫秒数再加上延迟(Delay)时间，如果你调用的不是sendMessageDelayed()方法，延迟时间就为0，然后将这两个参数都传递到MessageQueue的enqueueMessage()方法中。MessageQueue就是我们说的消息队列啦，由于MessageQueue是在Looper中创建，所以一个线程中只有一个MessageQueue。

OK,现在我们的消息已经要如队列了，我们看看这个MessageQueue中的enqueueMessage()方法实现。

{%highlight java%}

    boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {//msg.when != 0
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }

        synchronized (this) {
            if (mQuitting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }

            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);//本地方法，MessageQueue本质是由JNI层实现
            }
        }
        return true;
    }
{%endhighlight%}

MessageQueue只使用了一个mMessages对象表示当前待处理的消息。分析以上代码可知：我们的msg入队的时候，如果当前MessageQueue中的mMessages对象不为空，说明有某个message正在入队，此时在for循环中等待，一直等到mMessages(也就是p)为空，此时将我们的msg插入队列中。其实所谓的入队就是将所有的消息按时间来进行排序，这个时间当然就是我们刚才介绍的uptimeMillis参数。

如果你是通过sendMessageAtFrontOfQueue()方法来发送消息的，它也会调用enqueueMessage()来让消息入队，只不过时间为0，这时会把mMessages赋值为新入队的这条消息，然后将这条消息的next指定为刚才的mMessages，这样也就完成了添加消息到队列头部的操作。

我们的msg消息已经入队了，那出对怎么出？（参见一开始的那幅图）
这个秘密就在Looper.loop()中。

{%highlight java%}
public static void loop() {
        final Looper me = myLooper();//从ThreadLocal中获取Looper
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;//获取消息队列
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        //重点就在这个for循环
        for (;;) {
        	//我们的msg消息取出来了
        	//当前MessageQueue中存在mMessages(即待处理消息)，
        	//就将这个消息出队，然后让下一条消息成为mMessages，
        	//否则就进入一个阻塞状态，一直等到有新的消息入队
            Message msg = queue.next(); // might block
            if (msg == null) {
                // 取出的消息为null，说明消息队列已退出(异常或其他)
                //结束for循环 loop也退出
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            //分发消息到指定的Handler,msg.target其实就是Handler
            //sendMessageAtTime中指定msg.target
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycle();
        }
    }
    /*如果mCallback不为空，则调用mCallback的handleMessage()
    *方法，否则直接调用Handler的handleMessage()方法，并将消息对象作为
    *参数传递过去。这样我相信大家就都明白了为什么handleMessage()
    *方法中可以获取到之前发送的消息了吧！
    */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {//callback，下面会解释
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
{%endhighlight%}

因此，一个最标准的异步消息处理线程的写法应分为四步：

{%highlight java%}
class LooperThread extends Thread {
      public Handler mHandler;//1.定义Handler
      public void run() {
          Looper.prepare();//2.初始化Looper

          mHandler = new Handler() {//3.定义处理消息的方法
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();//4.启动消息循环
      }
  }
{%endhighlight%}

总结一下：由于Handler总是依附于创建时所在的线程，比如我们的Handler是在主线程中创建的，而在子线程中又无法直接对UI进行操作，于是我们就通过一系列的发送消息、入队、出队等环节，最后调用到了Handler的handleMessage()方法中，这时的handleMessage()方法已经是在主线程中运行的，因而我们当然可以在这里进行UI操作了（对应上面的图应该理解的更清晰）。

还有一个问题，上上面代码中的callback是什么？

原来在View.post(Runnable r)中使用到的，View.post会调用attachInfo.mHandler.post(action),我们来看一下Handler中的post：

{%highlight java%}
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;//原来这个callback就是我们定义的Runnable任务呀！
        return m;
    }
{%endhighlight%}

在Handler的dispatchMessage()方法中要做一个检查，如果Message的callback等于null才会去调用handleMessage()方法，否则就调用handleCallback()方法。原来直接调用了Runnable对象的run方法。

{%highlight java%}
    private static void handleCallback(Message message) {
        message.callback.run();
    }
{%endhighlight%}

通过分析部分源码，不管是使用什么方法在子线程中更新UI，其实背后的原理都是相同的，必须都要借助异步消息处理的机制来实现，再一次感叹异步消息处理机制的精妙！
