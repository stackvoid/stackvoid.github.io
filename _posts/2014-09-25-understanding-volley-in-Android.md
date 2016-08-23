---
layout: post
title: 深入理解Volley源码
categories: [Android]
tags: [Volley]
---

上篇博客提到使用StringRequest或JSONRequest等需要如下几个步骤：

1. 创建一个RequestQueue。(假设我们创建了一个RequestQueue，名字为myRequestQueue)
1. 创建相应的XXXRequest。(假设我们创建了一个StringRequest，为myStringRequest)
1. 把XXXRequest放到RequestQueue中。(将myStringRequest放到myRequestQueue中)

下面我们就行这三个步骤来分析一下Volley部分源码，看看内部究竟发生了什么
好玩的事情。

创建RequestQueue是在Volley.java中，我们来看看源代码：

{% highlight java %}
public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();//这里是关键，我们下面看一下RequestQueue的start方法

        return queue;
    }
    public static RequestQueue newRequestQueue(Context context) {
        return newRequestQueue(context, null);//这个是我们常用的RequestQueue，会调用到上面的方法
    }
{% endhighlight %}

通常我们使用newRequestQueue(Context context)获得RequestQueue对象，调用start()后，
返还给我们RequestQueue对象。那start方法做了什么呢？

{% highlight java %}
public void stop() {
        if (mCacheDispatcher != null) {
            mCacheDispatcher.quit();
        }
        for (int i = 0; i < mDispatchers.length; i++) {
            if (mDispatchers[i] != null) {
                mDispatchers[i].quit();
            }
        }
    }
public void start() {
        stop();  停掉所有的 cache 和 network dispatchers
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
{% endhighlight %}

原来start()首先停掉所有的CacheDispatcher和NetworkDispatcher，一般来说启动start()方法前都是没运行的(没还初始化)，
**这里为什么要先用stop()方法呢？，感觉有点多余啊！**
然后建立CacheDispatcher对象，新建4个(默认)NetworkDispatcher线程用来承载网络请求。下面我们来分析
CacheDispatcher和NetworkDispatcher。

###CacheDispatcher

在上面的start()中，新建的CacheDispatcher对象被直接start()了，CacheDispatcher继承自Thread，所有我们来分析一下
CacheDispatcher的run()方法。

{% highlight java %}
 public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
		//设定此CacheDispatcher线程为后台线程
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
		//Cache是一个接口，定义自己的Cache时，实现initialize方法来完成一些初始化
        mCache.initialize();

        while (true) {
            try {
				//mCacheQueue是BlockingQueue，当前队列没有Request，就阻塞在这里
				//mCacheQueue里有Request时，取出这个Request
                final Request request = mCacheQueue.take();
                request.addMarker("cache-queue-take");

                //Request已经被取消，本次循环结束
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                //看看在我们的Cache里面有没有这个Request
                Cache.Entry entry = mCache.get(request.getCacheKey());
                if (entry == null) {//没有的话直接放到网络请求队列mNetworkQueue中
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                if (entry.isExpired()) {
				//这个Request过期了，更新这个Request，并放到mNetworkQueue中
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");//命中Cache中的数据
				//解析NetworkResponse数据，我们自定义Request的时候，就要重写这个方法
				//顾名思义，就是拿到网络数据了，应该怎么解析。
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");
				//如果请求有效且并不需要刷新，则让Delivery中处理，
				//最终会触发如StringRequest这样的请求子类的onResponse或onErrorResponse  
                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {//请求有效，但要刷新，直接放到网络请求队列mNetworkQueue中 
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {//放入网络请求队列中刷新此Request
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
}

{% endhighlight %}

OK，到这里，我们的myRequestQueue终于返回来了，也就是第一步已经完成了。

第二步是定义StringRequest(以这个举例)，这个在NetworkDispatcher讲完后，你就明白里面定义参数的意义了。

第三步是将myStringRequest对象通过add()放到myRequestQueue中去。
我们先来看一下add方法然后再分析NetworkDispatcher。

{% highlight java %}
public Request add(Request request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);//方便Request完成后回调
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);//将此Request加入这个HashSet中
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {//Request不需要缓存，直接放入到网络请求队列mNetworkQueue中
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();//这个Key为Request请求的URL
            if (mWaitingRequests.containsKey(cacheKey)) {
				//mWaitingRequests(HashMap)中已经有此key
                //我们创建一个链表，存放第二个到达的Request
                Queue<Request> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {//说明这个Request是第二个同Key的Request
                    stagedRequests = new LinkedList<Request>();
                }
                stagedRequests.add(request);
				//更新这个key的Value
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {//没有这个key，说明这个Request是新的，将value置为null
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
{% endhighlight %}

我们来小结一下add方法所做的事情：

1. 我们的Request首先会加入到RequestQueue实体类的mCurrentRequests(HashSet)中做本地管理.
1. 我们这个Request如果是新请求，则保存到mWaitingRequests(HashMap)中(Key为我们Request的URL，Value为null)；如果不是新请求，则新建一个链表作为Value，同样Key为我们Request的URL
1. 分析到这里，我们已经将我们的Request放到mWaitingRequests中了，下面的介绍，会有线程去取这个Request。

###NetworkDispatcher

NetworkDispatcher在RequestQueue建立的时候，被start()所触发，默认建立4个NetworkDispatcher线程，并通过NetworkDispatcher的start方法触发此线程。
需要注意的是，NetworkDispatcher构造方法中的queue为RequestQueue中的mNetworkQueue，和CacheDispatcher用的是相同的。
mCache, mDelivery都是跟CacheDispatcher用的是相同的，这样方便数据(Request)共享。mDelivery为最终的消息配发者。
{% highlight java %}
public NetworkDispatcher(BlockingQueue<Request<?>> queue,  
        Network network, Cache cache,  
        ResponseDelivery delivery) {  
    mQueue = queue;  
    mNetwork = network;  
    mCache = cache;  
    mDelivery = delivery;  
}  
{% endhighlight %}

解析来分析必然看一下NetworkDispatcher的run()方法。

{% highlight java %}
public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request request;
        while (true) {
            try {
                // Take a request from the queue.
				//CacheDispatcher加进来的请求是从这里被NetworkDispatcher获取到的  
                request = mQueue.take();//这个Queue就是mNetworkQueue，是阻塞的
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {//这个Request被取消
                    request.finish("network-discard-cancelled");
                    continue;
                }

                // Tag the request (if API >= 14)
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                    TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
                }

                // Perform the network request.
				//使用BasicNetwork处理请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
				//又到这里了(CacheDispatcher的Request曾从缓存取出)，使用这个方法处理网络请求应答networkResponse 
				//NetworkResponse保存了请求的回应数据，包括数据本身和头，还有状态码以及其他相关信息。
				//NetworkResponse根据请求类型的不同，对回应数据的处理方式也各有不同
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();//标记这个Request的消息已经被分发了
                mDelivery.postResponse(request, response);//消息分发
            } catch (VolleyError volleyError) {
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                mDelivery.postError(request, new VolleyError(e));//未知错误，只会触发mDelivery的postError方法
            }
        }
    }

{% endhighlight %}

我们来小结一下NetworkDispatcher所做的事情：

1. 首先从mNetworkQueue中取出一个Request，如果没有Request，则阻塞一直到有。
1. 若此Request被取消，则取下一个Request。
1. 使用BasicNetwork的performRequest()处理此Request，并返回我们想要的数据，保存在NetworkResponse对象中。
1. NetworkResponse根据Request类型(如StringRequest、JSONRequest)的不同，对回应数据的处理方式(parseNetworkResponse())也各有不同
1. 回调Request，调用我们自己定义的onResponse(XXX xxx)
1. 使用mDelivery.postResponse(request, response)，消息分发

BasicNetwork的performRequest()方法使用HttpStack来完成网络数据获取任务(查看BasicNetwork类)，本文不做详细介绍了。

下面我们来看 parseNetworkResponse方法对NetworkResponse的处理。由于这个方法是Request自己提供的，不同Request类型，此方法也不一样；我们拿StringRequest来举例。
在StringRequest.java中。
{% highlight java %}
protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }

public class NetworkResponse {
...........

    /** The HTTP status code. */
    public final int statusCode;

    /** Raw data from this response. */
    public final byte[] data;

    /** Response headers. */
    public final Map<String, String> headers;

    /** True if the server returned a 304 (Not Modified). */
    public final boolean notModified;
}
{% endhighlight %}

我们看到，在StringRequest的parseNetworkResponse中，成功后将返回Response.success(...)。

我们再来看一下Response类。
{% highlight java %}
public class Response<T> {

    /** Callback interface for delivering parsed responses. */
    public interface Listener<T> {
        /** Called when a response is received. */
        public void onResponse(T response);//这不就是我们覆写的方法嘛~！！
    }

    /** Callback interface for delivering error responses. */
    public interface ErrorListener {
        /**
         * Callback method that an error has been occurred with the
         * provided error code and optional user-readable message.
         */
        public void onErrorResponse(VolleyError error);//覆写产生ERROR的方法
    }

    /** Returns a successful response containing the parsed result. */
    public static <T> Response<T> success(T result, Cache.Entry cacheEntry) {
        return new Response<T>(result, cacheEntry);
    }
{% endhighlight %}

到这里，我们就明白了，原来parseNetworkResponse将数据(也就是NetworkResponse的data数据)封装成Response对象。
耦合关系太巧妙了！！

分析到这里，好像已经分析完了，等等！我们自定义的onResponse方法还没执行呢！NetworkDispatcher的run()方法还有最后一步：mDelivery.postResponse(request, response)

这个是做什么的呢？听我一一道来。

我们这里还是以StringResponse为例介绍。

{% highlight java %}
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {//错误响应
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }
{% endhighlight %}

重点是ResponseDeliveryRunnable这个类了，我们来分析一下这个类。

{% highlight java %}
//内部类ResponseDeliveryRunnable
public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {//若请求被取消，结束并做标记
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {//请求成功，调用我们Request的deliverResponse来响应
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");//清除自己
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
	   
//in StringRequest.java
@Override  
protected void deliverResponse(String response) {  
    mListener.onResponse(response);  //调用了我们覆写的onResponse方法。
}  
{% endhighlight %}
我们看到，如果mResponse.isSuccess是成功的，则调用我们Request的deliverResponse方法，间接调用了我们自己覆写的onResponse方法。
分析到这里，我们覆写的onResponse都调用完了，好像哪里还没做完，原来是RequestQueue中的几个集合，我们还没把我们执行完的Request在集合中做处理。

我们最后分析一下清除此Request的mRequest.finish("done");

{% highlight java %}
void finish(final String tag) {
        if (mRequestQueue != null) {
            mRequestQueue.finish(this);//若请求队列有效，则在请求队列中标记当前Request为结束
        }
..........
    }
//RequestQueue的finish方法
	    void finish(Request request) {
        // Remove from the set of requests currently being processed.
        synchronized (mCurrentRequests) {
            mCurrentRequests.remove(request);//从本地集合中删除这个Request
        }

        if (request.shouldCache()) {//这个Request是否需要缓存
            synchronized (mWaitingRequests) {
                String cacheKey = request.getCacheKey();//获取cacheKey，即URL 
                Queue<Request> waitingRequests = mWaitingRequests.remove(cacheKey);
                if (waitingRequests != null) {//可能有一个LinkedList存相同的Key的Value
                    if (VolleyLog.DEBUG) {
                        VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                                waitingRequests.size(), cacheKey);
                    }
                    // Process all queued up requests. They won't be considered as in flight, but
                    // that's not a problem as the cache has been primed by 'request'.
                    mCacheQueue.addAll(waitingRequests);//将其他的请求全部加入到mCacheQueue中交由CacheDispatcher处理  
                }
            }
        }
    }
{% endhighlight %}

OK, 分析到这里，我们队Volley了解是不是有了本质的提升，不得不感叹设计Volley架构的工程师，将这么多复杂的逻辑屏蔽，
仅仅用简单方法入口，就能使用这么多强大的网络功能~！
