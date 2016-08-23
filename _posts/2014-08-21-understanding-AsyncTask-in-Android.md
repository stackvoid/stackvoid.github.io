---
layout: post
title: 深入理解AsyncTask
categories: [Android]
tags: [Async]
---

开发Android app的时候通常将耗时的操作放在单独的线程中执行，避免其占用主线程(主要负责更新UI)而给用户带来不良用户体验。所以Android提供了一个Handler类在子线程完成任务后异步通知UI线程，主线程(UI线程)收到消息更新UI界面，呈现给用户。比较好的解决了子线程更新UI的问题。但是费时的任务操作总会启动一些匿名的子线程，太多的子线程给系统带来巨大的负担，随之带来一些性能问题。因此Android提供了一个工具类AsyncTask。这个AsyncTask生来就是处理一些后台的比较耗时的任务，给用户带来良好用户体验的，从编程的语法上显得优雅了许多，不再处理子线程和Handler(AsyncTask源码最终还是使用Handler机制)就可以完成异步操作并且刷新用户界面。

##使用方法

AsyncTask是一个抽象类，必须要创建一个子类去继承它才可以使用。在继承时我们可以为AsyncTask类指定三个泛型参数：

- Params：这个泛型指定在执行AsyncTask时需要传入的参数，可用于在后台任务中使用；比如HTTP请求的URL。
- Progress：异步任务在执行的时候将执行的进度返回给UI线程的参数的类型，则使用这里指定的泛型作为进度单位。
- Result：异步任务执行完后返回给UI线程的结果的类型。

###使用AsyncTask的一般步骤

本文中的实例 DownLoadTask 来自Android官方文档。

**1.继承AsyncTask**

{%highlight java%}
//无传入参数
//使用整型数据来作为进度显示单位
//任务执行完后返回值为Boolean类型
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {
	……
}
{%endhighlight%}

**2.重写AsyncTask中的方法**

重写AsyncTask中的以下几个方法才能完成对任务的定制。

- onPreExecute(): 这个方法是在执行异步任务之前的时候执行，并且是在UI Thread当中执行的，通常我们在这个方法里做一些UI控件的初始化的操作，比如显示一个进度条对话框或一些控件的实例化等。
- doInBackground(Params...)：这个方法中的所有代码都会在子线程中运行，应在这里处理耗时任务。任务一旦完成就可以通过return语句来将任务的执行结果进行返回，如果AsyncTask的第三个泛型参数指定的是Void，就可以不返回任务执行结果。注意，在这个方法中是**不可以进行UI操作**(因为在子线程)，如果需要更新UI元素，比如说反馈当前任务的执行进度，可以调用publishProgress(Progress...)方法来完成。这个方法子类**必须重写**。
- onProgressUpdate(Progress...)：在UI Thread中执行的，在异步任务执行的时候，有时需要将执行的进度返回给我们的UI界面，例如下载一张网络图片，我们需要时刻显示其下载的进度，就可以使用这个方法来更新我们的进度。这个方法在调用之前，我们需要在 doInBackground 方法中调用一个 publishProgress(Progress) 的方法来将我们的进度时时刻刻传递给 onProgressUpdate 方法来更新。
- onPostExecute(Result)：在doInBackground 执行完成后，onPostExecute 方法将被UI thread调用，可以将返回的结果显示在UI控件上。
- onCancelled()：在用户取消线程操作的时候调用。在主线程中调用onCancelled()的时候调用。

一个改自Android文档上的例子。

{%highlight java%}
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
	@Override
	protected void onPreExecute() {
		progressDialog.show();
	}
	@Override
     protected Long doInBackground(URL... urls) {
         int count = urls.length;
         long totalSize = 0;
         for (int i = 0; i < count; i++) {
             totalSize += Downloader.downloadFile(urls[i]);
             publishProgress((int) ((i / (float) count) * 100));
             // Escape early if cancel() is called
             if (isCancelled()) break;
         }
         return totalSize;
     }
	@Override
     protected void onProgressUpdate(Integer... values) {
         progressDialog.setMessage("当前下载进度：" + values[0] + "%");
     }
     @Override
     protected void onPostExecute(Long result) {
        progressDialog.dismiss();
		if (result) {
			Toast.makeText(context, "下载成功", Toast.LENGTH_SHORT).show();
		} else {
			Toast.makeText(context, "下载失败", Toast.LENGTH_SHORT).show();
		}
 }
{%endhighlight%}
我们模拟了一个下载任务，在doInBackground()方法中去执行具体的下载逻辑，在onProgressUpdate()方法中显示当前的下载进度，在onPostExecute()方法中来提示任务的执行结果。简单地调用以下代码即可启动此任务：

{%highlight java%}
new DownloadTask().execute(); 
{%endhighlight%}

###使用AsyncTask必须要遵循的原则

- AsyncTask的对象必须在UI Thread当中实例化
- execute方法必须在UI Thread当中调用
- 请勿手动的去调用AsyncTask的onPreExecute, doInBackground, publishProgress, onProgressUpdate, onPostExecute方法，这些都是由Android系统自动调用的
- AsyncTask任务只能被执行一次，即只能调用一次execute方法，多次调用时将会出现异常
- AsyncTask不是被设计为解决非常耗时操作的，耗时上限为几秒钟，如果要做长耗时操作，强烈建议使用Executor，ThreadPoolExecutor以及FutureTask。

**注意**AsyncTask不能完全取代线程，在一些逻辑较为复杂或者需要在后台反复执行的逻辑就可能需要线程来实现了。


##AsyncTask的源码解析

AsyncTask的源码基于Android4.4.4，其实源码文档已经说得很细了，本文分析源码做下补充。

AsyncTask也是使用的异步消息处理机制，只是做了非常好的封装而已。


{%highlight java%}
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";
    //获取当前机器的cpu核心数
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    //线程池推荐容量，一般都为处理器核心数+1(更好利用多核)
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    //线程池最大容量，容量过大，线程切换开销总和也很大
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    //过剩的空闲线程的存活时间
    private static final int KEEP_ALIVE = 1;

    //工厂方法模式，通过newThread来获取新线程
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    	//原子整数，高并发下正常工作无压力
        private final AtomicInteger mCount = new AtomicInteger(1);
        //获取新建线程
        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    //静态阻塞式队列，用来存放待执行的任务，初始容量：128个
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * 静态并发线程池，可以用来并行执行任务，AsyncTask默认是串行执行任务
	 * 但仍能构造出并行的AsyncTask
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * 内部实现了串行控制，循环的取出一个个任务交给上述的并发线程池去执行
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    //消息类型：发送结果
    private static final int MESSAGE_POST_RESULT = 0x1;
    //消息类型：更新进度
    private static final int MESSAGE_POST_PROGRESS = 0x2;
	/*静态Handler，用来发送上述两种通知，采用UI线程的Looper来处理消息
	 * 这就是为什么AsyncTask必须在UI线程调用，因为子线程
	 * 默认没有Looper无法创建下面的Handler，程序会直接Crash
	 */
    private static final InternalHandler sHandler = new InternalHandler();
    //默认任务执行器，被赋值为串行任务执行器，就是它，AsyncTask变成串行的了
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
	//任务的状态 默认为挂起, volatile经典用法之一：当做标识
    private volatile Status mStatus = Status.PENDING;
    //标识任务是否被取消
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    //标识任务是否被执行过
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
/*串行执行器的实现,asyncTask.execute(Params ...)实际上会调用
 *SerialExecutor的execute方法;即当你的asyncTask执行的时候，
 *首先你的task会被加入到任务队列，然后排队，一个个执行
 */
    private static class SerialExecutor implements Executor {
        //线性双向队列，用来存储所有的AsyncTask任务
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        //当前正在执行的AsyncTask任务
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            ////将新的AsyncTask任务加入到双向队列中
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
						//执行AsyncTask任务
						//其实为mFuture对象，调用mFuture对象的run方法
						//run调用Sync内部类的innerRun()，在调用call()方法
						//call方法里有postResult(doInBackground(mParams));
						//所以doInBackground最终是在这里执行(毕竟新开启了一个线程)
                        r.run();
                    } finally {
                    	//当前AsyncTask任务执行完毕后，如果还有未执行任务 则进行下一轮执行， 
                        //明显体现AsyncTask是串行执行任务，总是一个任务执行完毕才会执行下一个任务 
                        scheduleNext();
                    }
                }
            });
            //第一次运行当然是等于null了，于是会调用scheduleNext()方法
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
        	//从任务队列中取出队列头部的任务，如果有就交给并发线程池去执行
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    /**
     * 任务的三种状态,在整个Task的生命周期中，只能被分别设置一次
     */
    public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         */
        PENDING,
        /**
         * Indicates that the task is running.
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         */
        FINISHED,
    }

    /** @hide 在UI线程中调用，用来初始化Handler  */
    public static void init() {
        sHandler.getLooper();
    }

    /** @hide 为AsyncTask设置默认执行器*/
    public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
	//doInBackground执行完毕，发送消息
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

    /**
     * 返回任务的状态
     *
     * @return The current status.
     */
    public final Status getStatus() {
        return mStatus;
    }

    /**
     * 这个方法是我们必须要重写的，用来做后台计算
     * 所在线程：后台线程
     * This method can call {@link #publishProgress} to publish updates
     * on the UI thread.
     *
     * @param params The parameters of the task.
     *
     * @return A result, defined by the subclass of this task.
     *
     * @see #onPreExecute()
     * @see #onPostExecute
     * @see #publishProgress
     */
    protected abstract Result doInBackground(Params... params);

    /**
     * R在doInBackground之前调用，用来做初始化工作 
     * 所在线程：UI线程
     * @see #onPostExecute
     * @see #doInBackground
     */
    protected void onPreExecute() {
    }

    /**
     * 在doInBackground之后调用，用来接受后台计算结果更新UI 
     * 所在线程：UI线程 
     *
     * @param result The result of the operation computed by {@link #doInBackground}.
     *
     * @see #onPreExecute
     * @see #doInBackground
     * @see #onCancelled(Object) 
     */
    @SuppressWarnings({"UnusedDeclaration"})
    protected void onPostExecute(Result result) {
    }

    /**
     * 在publishProgress之后调用，用来更新计算进度
     * 所在线程：UI线程 
     *
     * @param values The values indicating progress.
     *
     * @see #publishProgress
     * @see #doInBackground
     */
    @SuppressWarnings({"UnusedDeclaration"})
    protected void onProgressUpdate(Progress... values) {
    }

    /**
     * cancel被调用并且doInBackground执行结束，会调用onCancelled，表示任务被取消 
     *这个时候onPostExecute不会再被调用，二者是互斥的，分别表示任务取消和任务执行完成 
     * 所在线程：UI线程
     * @param result The result, if any, computed in
     *               {@link #doInBackground(Object[])}, can be null
     * 
     * @see #cancel(boolean)
     * @see #isCancelled()
     */
    @SuppressWarnings({"UnusedParameters"})
    protected void onCancelled(Result result) {
        onCancelled();
    }    
    
    /**
     * <p>Applications should preferably override {@link #onCancelled(Object)}.
     * This method is invoked by the default implementation of
     * {@link #onCancelled(Object)}.</p>
     * 
     * <p>Runs on the UI thread after {@link #cancel(boolean)} is invoked and
     * {@link #doInBackground(Object[])} has finished.</p>
     *
     * @see #onCancelled(Object) 
     * @see #cancel(boolean)
     * @see #isCancelled()
     */
    protected void onCancelled() {
    }

    /**
     * Returns <tt>true</tt> if this task was cancelled before it completed
     * normally. If you are calling {@link #cancel(boolean)} on the task,
     * the value returned by this method should be checked periodically from
     * {@link #doInBackground(Object[])} to end the task as soon as possible.
     *
     * @return <tt>true</tt> if task was cancelled before it completed
     *
     * @see #cancel(boolean)
     */
    public final boolean isCancelled() {
        return mCancelled.get();
    }

    /**
     * <p>Attempts to cancel execution of this task.  This attempt will
     * fail if the task has already completed, already been cancelled,
     * or could not be cancelled for some other reason. If successful,
     * and this task has not started when <tt>cancel</tt> is called,
     * this task should never run. If the task has already started,
     * then the <tt>mayInterruptIfRunning</tt> parameter determines
     * whether the thread executing this task should be interrupted in
     * an attempt to stop the task.</p>
     * 
     * <p>Calling this method will result in {@link #onCancelled(Object)} being
     * invoked on the UI thread after {@link #doInBackground(Object[])}
     * returns. Calling this method guarantees that {@link #onPostExecute(Object)}
     * is never invoked. After invoking this method, you should check the
     * value returned by {@link #isCancelled()} periodically from
     * {@link #doInBackground(Object[])} to finish the task as early as
     * possible.</p>
     *
     * @param mayInterruptIfRunning <tt>true</tt> if the thread executing this
     *        task should be interrupted; otherwise, in-progress tasks are allowed
     *        to complete.
     *
     * @return <tt>false</tt> if the task could not be cancelled,
     *         typically because it has already completed normally;
     *         <tt>true</tt> otherwise
     *
     * @see #isCancelled()
     * @see #onCancelled(Object)
     */
    public final boolean cancel(boolean mayInterruptIfRunning) {
        mCancelled.set(true);
        return mFuture.cancel(mayInterruptIfRunning);
    }

    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     *
     * @return The computed result.
     *
     * @throws CancellationException If the computation was cancelled.
     * @throws ExecutionException If the computation threw an exception.
     * @throws InterruptedException If the current thread was interrupted
     *         while waiting.
     */
    public final Result get() throws InterruptedException, ExecutionException {
        return mFuture.get();
    }

    /**
     * Waits if necessary for at most the given time for the computation
     * to complete, and then retrieves its result.
     *
     * @param timeout Time to wait before cancelling the operation.
     * @param unit The time unit for the timeout.
     *
     * @return The computed result.
     *
     * @throws CancellationException If the computation was cancelled.
     * @throws ExecutionException If the computation threw an exception.
     * @throws InterruptedException If the current thread was interrupted
     *         while waiting.
     * @throws TimeoutException If the wait timed out.
     */
    public final Result get(long timeout, TimeUnit unit) throws InterruptedException,
            ExecutionException, TimeoutException {
        return mFuture.get(timeout, unit);
    }

    /**
     *这个方法如何执行和系统版本有关，在AsyncTask的使用规则里已经说明，如果你真的想使用并行AsyncTask， 
     * 也是可以的，只要返回 THREAD_POOL_EXECUTOR 即可
     * 必须在UI线程调用此方法 
     *
     * @param params The parameters of the task.
     *
     * @return This instance of AsyncTask.
     *
     * @throws IllegalStateException If {@link #getStatus()} returns either
     *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
     *
     * @see #executeOnExecutor(java.util.concurrent.Executor, Object[])
     * @see #execute(Runnable)
     */
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    	//默认串行执行
        return executeOnExecutor(sDefaultExecutor, params);
        //如果我们想并行执行，返回THREAD_POOL_EXECUTOR即可  
        //return executeOnExecutor(THREAD_POOL_EXECUTOR, params);  
    }

   /** 
     * 通过这个方法我们可以自定义AsyncTask的执行方式，串行or并行，
     * 甚至可以采用自己的Executor 
     * 为了实现并行，我们可以在外部这么用AsyncTask： 
     * asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, Params... params); 
     * 必须在UI线程调用此方法 
     */  
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;
        //这里#onPreExecute会最先执行 
        onPreExecute();

        mWorker.mParams = params;
        //然后后台计算#doInBackground才真正开始  
        exec.execute(mFuture);//接着会有#onProgressUpdate被调用，最后是#onPostExecute 

        return this;
    }

	//方便我们直接执行一个runnable 
    public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }

	//打印后台计算进度，onProgressUpdate会被调用
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            sHandler.obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
         //任务结束的时候会进行判断，如果任务没有被取消，则onPostExecute会被调用  
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    //说明任务执行完成，调用finish结束任务
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
}
{%endhighlight%}
