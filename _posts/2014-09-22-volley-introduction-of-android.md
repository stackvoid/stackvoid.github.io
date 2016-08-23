---
layout: post
title: Android网络工具Volley(应用篇)
categories: [Android]
tags: [Volley]
---

Volley之前，Android系统中主要提供了两种方式来进行HTTP通信，HttpURLConnection和HttpClient，我们几乎在任何项目的代码中都能看到这两个类的身影，出境率非常高。但二者使用起来十分不方便，尤其大量连接访问网络数据时，重复写很多代码，于是Volley出现了。Volley非常适合数据量不大但是访问网络频繁的应用。

本文主要介绍Volley工具的用法，包括：StringRequest、JsonRequest(JsonObjectRequest和JsonArrayRequest)、ImageRequest、ImageLoader和NetworkImageView用法。

###Volley获取与编译

在Googlesource上获取Volley最新源码：

*git clone https://android.googlesource.com/platform/frameworks/volley*

如果有完整的Android SDK，把它放到Frameworks文件夹里，到Volley目录下，直接用mm命令编译即可。

若没有完整的SDK，用 ant jar 编译即可。编译出来直接扔到工程libs即可。

###StringRequest用法


1. 创建一个网络请求队列RequestQueue
1. 创建StringRequest对象
1. 将此StringRequest对象添加到RequestQueue中


我们先使用GET方式来获取百度的页面。
{% highlight java %}
RequestQueue mQueue = Volley.newRequestQueue(getApplicationContext()); 

StringRequest stringRequest = new StringRequest("http://www.baidu.com",
						new Response.Listener<String>() {
							@Override
							public void onResponse(String response) {
								//成功应答后处理返回的response
							}
						}, new Response.ErrorListener() {
							@Override
							public void onErrorResponse(VolleyError error) {
								//出错时的措施
								Log.e("TAG", error.getMessage(), error);
							}
						});
//最后将这个StringRequest扔到我们的请求队列中去
mQueue.add(mQueue);
{% endhighlight %}

StringRequest还提供了POST方式来请求或提交数据。请求服务器数据的POST和GET方式差不多，下面说一下用POST提交数据。

{% highlight java %}
StringRequest request = new StringRequest(  
                Request.Method.POST,  
                someURL,  
                new Response.Listener<String>() {  
                    @Override  
                    public void onResponse(String s) {  
  
                    }  
                },  
                new Response.ErrorListener() {  
                    @Override  
                    public void onErrorResponse(VolleyError volleyError) {  
  
                    }  
                }  
        ) {  
  
            @Override  
            protected Map<String, String> getParams() throws AuthFailureError {  //设置参数  
                Map<String, String> map = new HashMap<String, String>();  
                map.put("name", "foo");  
                map.put("password", "123");  
                return map;  
            }  
        };  
{% endhighlight %}

###JsonRequest

JsonRequest是抽象类继承于Request，有两个子类分别是JsonObjectRequest和JsonArrayRequest。

JSONArrayRequest接受JSONArray，不支持JSONObject，且只支持GET。

{% highlight java %}
final String URL = "stackvoid.com/cn";
JsonArrayRequest req = new JsonArrayRequest(URL, new Response.Listener<JSONArray> () {
    @Override
    public void onResponse(JSONArray response) {
        try {
            VolleyLog.v("Response:%n %s", response.toString());
        } catch (JSONException e) {
            e.printStackTrace();
        }
    }
}, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError error) {
        VolleyLog.e("Error: ", error.getMessage());
    }
});
mQueue.add(...);
{% endhighlight %}
 JSONObjectRequest使用类似，其可以使用GET也可以使用POST，不在讲述。


### ImageRequest

使用ImageRequest跟上两个一样，都需要三步走。

{% highlight java %}
RequestQueue mQueue = Volley.newRequestQueue(context); 

/*ImageRequest的构造函数接收六个参数
*第一个参数就是图片的URL地址.
*第二个参数是图片请求成功的回调.
*第三第四个参数分别用于指定允许图片最大的宽度和高度，如果指定的网络图片
*的宽度或高度大于这里的最大值，则会对图片进行压缩，指定成0的话就表示不管
*图片有多大，都不会进行压缩.
*第五个参数用于指定图片的颜色属性.
*第六个参数是图片请求失败的回调.
*/
ImageRequest imageRequest = new ImageRequest(
		"http://developer.android.com/images/home/aw_dac.png",
		new Response.Listener<Bitmap>() {
			@Override
			public void onResponse(Bitmap response) {
				imageView.setImageBitmap(response);//imageView是控件
			}
		}, 0, 0, Config.RGB_565, new Response.ErrorListener() {
			@Override
			public void onErrorResponse(VolleyError error) {
				imageView.setImageResource(R.drawable.default_image);
			}
		});
mQueue.add(imageRequest);
{% endhighlight %}

**更好用的ImageLoader**

ImageLoader也可用于加载网络上的图片，内部也使用ImageRequest来实现，不过ImageLoader明显要比ImageRequest更加高效，因为它不仅可以帮我们对图片进行缓存，还可以过滤掉重复的链接，避免重复发送请求。

下面我们结合者代码学习使用ImageLoader。

{% highlight java %}
//1. 创建一个RequestQueue
RequestQueue mQueue = Volley.newRequestQueue(getContext());
//2. 创建一个imageLoader,cache是缓存
//要写一个性能非常好的ImageCache，最好就要借助Android提供的LruCache功能
//然后实现ImageCache即可
ImageLoader imageLoader = new ImageLoader(mQueue, new ImageCache() {
	@Override
	public void putBitmap(String url, Bitmap bitmap) {
	}

	@Override
	public Bitmap getBitmap(String url) {
		return null;
	}
});
//3. 获取一个ImageListener对象
//getImageListener()方法接收三个参数，第一个参数指定用于显示图片的ImageView控件，
//第二个参数指定加载图片的过程中显示的图片，
//第三个参数指定加载图片失败的情况下显示的图片。
ImageListener listener = ImageLoader.getImageListener(imageView,
		R.drawable.default_image, R.drawable.failed_image);

//4. 调用ImageLoader的get()方法加载网络上的图片
imageLoader.get(URL, listener);
{% endhighlight %}

###NetworkImageView的用法

NetworkImageView是继承自ImageView的，具备ImageView控件的所有功能，并且在原生的基础之上加入了加载网络图片的功能。

加入NetworkImageView控件：
{% highlight xml %}
........
<com.android.volley.toolbox.NetworkImageView 
       android:id="@+id/network_image_view"
       android:layout_width="200dp"
       android:layout_height="200dp"
       android:layout_gravity="center_horizontal"
       />
{% endhighlight %}

在Activity里获得NetworkImageView实例后，可调用它的setDefaultImageResId()方法、setErrorImageResId()方法和setImageUrl()方法来分别设置加载中显示的图片，加载失败时显示的图片，以及目标图片的URL地址。

{% highlight java %}
networkImageView.setDefaultImageResId(R.drawable.default_image);
networkImageView.setErrorImageResId(R.drawable.failed_image);
networkImageView.setImageUrl("www.stackvoid.com/icon",
				imageLoader);
{% endhighlight %}

上面代码中的imageLoader就是ImageLoader，参见上一小节即可。
