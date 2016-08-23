---
layout: post
title: Android应用中埋点监控的思考与设计
categories: [Android]
tags: [Monitor]
---
一款Android商业应用上线后，最关心的莫过于用户使用哪个模块比较频繁，哪个模块使用人群较少，产品可以根据这些数据来修正app以后的发展方向，使产生最大的商业价值。

通过埋点监控，我们可以深入业务的每一个细节，产生的用户行为可以通过所埋的点累计次数并将这些数据发送到数据中心，通过数据分析师就能给产品提出宝贵的意见，指导产品的演化方向。

本文基于我的上一篇博客[Android 事件分发机制详解](http://stackvoid.com/details-dispatch-onTouch-Event-in-Android/)，如果你对事件分发机制不是特别了解的话，建议先去看一下这篇文章。

###综述设计方案

我们的埋点方案要做到以下功能：

1. Android界面上的空间被用户点击，需要记录下点击控件的名称并保存此信息。
1. Android界面被打开或关闭，也需要记录此信息
1. 最好能自动化完成，不需要修改大量代码，最好能定制

设计思路大体如下： 

1. 设计一个基类BaseActivity，它是继承自Activity，但是覆写了Activity的几个方法(后面会详细说明)。
1. 利用广播来统一管理用户行为的Log信息。
1. 数据积累到一定量，将用户行为数据发送到后台服务器。

### BaseActivity基类的设计

利用Android事件分发机制，我们自定义的基类BaseActivity继承自Activity并重写Activity的dispatchTouchEvent方法(为什么要这么做？还请参考我的上一篇博客)，以及重写Activity的所有生命周期方法。

**重写Activity的生命周期以及事件分发方法**

重写Activity生命周期的onStart()和onStop(){或者onDestory，这个根据自己的选择确定}，来完成对界面开启和关闭的埋点记录。事件分发方法来检测ACTION_UP这个事件(也就是手指触动触摸屏抬起的那个事件)，二者通过本地广播，将onStart或onStop这些事件广播出来并被接收处理。

{% highlight java linenos %}
public class BaseActivity extends Activity {
	protected void onStart(){
		super.onStart();
		// 使用本地广播，高效更安全
		LoacalBroadcastManager bcManager = LocalBroadcastManager.getInstance(this);
		Intent intent = new Intent(ACTIVITY_START);//自定义的ACTIVITY_START
		bcManager.sendBroadcast(intent);
	}		
	protected void onStop(){
		super.onStop();
		LoacalBroadcastManager bcManager = LocalBroadcastManager.getInstance(this);
		Intent intent = new Intent(ACTIVITY_STOP);//自定义的ACTIVITY_STOP
		bcManager.sendBroadcast(intent);
	}
	//.......可扩展
	protected boolean dispatchTouchEvent(MotionEvent e){
		if (e.getAction() == MotionEvent.ACTION_UP){
			LocalBroadcastManager broadcastManager = LocalBroadcastManager.getInstance(this);
            Intent intent = new Intent(VIEW_CLICK);
            intent.putExtra(VIEW_CLICK, e);
            broadcastManager.sendBroadcast(intent);
		}
	}
{% endhighlight %}

**广播事件的处理**

在处理广播的事件类中，我们获得VIEW_CLICK的Action就开始遍历当前Activity中所有的View，通过比对点击事件event的坐标个View的坐标，来判断是点击哪个View的event。

{% highlight java linenos %}
public class BaseActivity extends Activity {
	//.......
	public class MonitorUserReceiver extends BroadcastReceiver {
		public void onReceive(Context context, Intent intent) {
			String action = intent.getAction();
			switch(action){
				case VIEW_CLICK:
					MotionEvent event = intent.getParcelableExtra(VIEW_CLICK);
					//递归遍历Activity中的所有View，找出被点击的View
					View clickView = searchClickView(view, event);
					//获取clickView的路径信息
					//生成log记录下来
					Log.writeLog();
					break;
				case ACTIVITY_START:
					//可以知道某个界面被打开了，然后记录此次操作行为
					Log.writeLog();
					break;
				case ACTIVITY_STOP:
					Log.writeLog();
					break;
				//可扩展...
			}
		}
		private View searchClickView(View view, MotionEvent event) {
			View clickView = null;
			if (isInView(view, event) && 
				view.getVisibility() == View.VISIBLE) {  //当前View必须是可见的
				if (view instanceof ViewGroup) {	//如果是类似Layout的ViewGroup，继续遍历它下面的子View
					ViewGroup group = (ViewGroup) view;
					for (int i = group.getChildCount() - 1; i >= 0; i--) {
						View childView = group.getChildAt(i);
						clickView = searchClickView(childView, event);
						if (clickView != null) {
							return clickView;
						}
					}
				}
				clickView = view;
			}	
			return clickView;
		}
		public boolean isInView(View view,MotionEvent event){
			int clickX = event.getRawX();	//获取点击事件的X和Y坐标
			int clickY = event.getRawY();
			//如下的view表示Activity中的子View或者控件
			int[] location = new int[2];	
			view.getLocationOnScreen(location);  
			int x = location[0];
			int y = location[1];
			int width = view.getWidth();
			int height = view.getHeight();
			if (clickX < x || clickX > (x + width) || 
				clickY < y || clickY > (y + height)) {
				return true;  //此条件成立，说明这个view被点击了
			}
			return false;
		}	
	}

		
{% endhighlight %}

**记录View的路径**

上面代码中提到要记录View的路径，我们可以通过给空间加Tag的方式，给此view空间位移的名字或ID，但一个Android app中的控件数量太多，想都加上Tag实在太麻烦，并且有漏加的风险。

Activity中的UI是层层嵌套的，其中根布局是PhoneWindow$DecorView，下面通过hierarchyviewer工具来举一个实例。

![monitorUser01](/album/2014-10-09-auto-monitor-user-behiver-in-android-app01.png)

上图有一个TextView，如果按照我的采用的是View控件的路径方式标识方法应该是：

DecorView[0]>ActionBarOverlayLayout[0]>FrameLayout[0]>RelativeLayout[0]>TextView
[0]":"helloworld"

在此路径前加上Activity的名字，便构成了控件View唯一的属性标识。例如我们在DemoActivity里有一个button，button名字为hello：

DemoActivity：DecorView[0]>ActionBarOverlayLayout[0]>FrameLayout[0]>RelativeLayout[0]>Button
[0]":"hello"

DecorView，可通过this.getWindow().getDecorView()获得。其实在searchClickView方法中我们可以加上路径，找到我们需要的View，那么当前这个路径自然就知道了。
**这种方法产生的问题有：**产生的路径是：

*DecorView>ActionBarOverlayLayout>FrameLayout>RelativeLayout>Button":"helloworld"*

当有一个重名的Button时，根本分不清楚到底是哪一个Button。要是我们能产生类似
*DemoActivity：DecorView[0]>ActionBarOverlayLayout[0]>FrameLayout[0]>RelativeLayout[0]>Button
[0]":"hello"*
这种路径，就能完美解决这个问题了。

最简单的解决办法就是使用 View 的 getParent(),不停调用，直到获取到 root view。例如下面的例子是要获取某个 Button的绝对路径：

{% highlight java linenos %}
Button startingButton = findViewById(R.id.startingButton);
ViewParent v = startingButton.getParent();
while(v != null) {
    System.out.println(String.valueOf(v.getId()) + " : " + v.getClass().getName());
    v = v.getParent();
}
{%endhighlight%}


**将用户行为信息传到后台服务器**

用户点击的log信息，我们可以用XML或JSON来格式化数据，然后存到app的目录下，一段时间后(这个自定义)开启新线程，将用户行为信息传送到后台服务器，这个步骤比较简单，就不上源码了。

###不足之处

1. 当前的APP使用这种*DemoActivity:DecorView>ActionBarOverlayLayout>FrameLayout>RelativeLayout>Button":"helloworld"*
绝对路径，尽管app初期比较简单，但是若有工程师不注意使用相同的名字控件，会出现找到第一个就返回情况，后期还要继续研究hierarchyviewer的源代码，找到使用绝对路径的方法。
1. 两个Button重叠，点击此Button，可能无法找到正确的一个，这个问题暂时没想出如何解决，只能靠工程师小心，不要加入重叠的Button。
