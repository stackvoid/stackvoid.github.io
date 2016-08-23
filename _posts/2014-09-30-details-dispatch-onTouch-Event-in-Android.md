---
layout: post
title: Android 事件分发机制详解
categories: [Android]
tags: [TouchEvent]
---

网上很多关于Android事件分发机制的解释，大多数描述的都不够清晰，没有吧来龙去脉搞清楚，本文将带你从Touch事件产生到Touch事件被消费这一全过程作全面的剖析。

###产生Touch事件

这部分牵扯到硬件和Linux内核部分；我们简单讲述一下这部分内容，如果有兴趣的话可以参考[这篇文章](http://stackoverflow.com/questions/3696103/android-pass-kernel-input-event-dev-input-event-to-java)。

###传递Touch事件

触摸事件是由Linux内核的一个Input子系统来管理的(InputManager)，Linux子系统会在 `/dev/input/` 这个路径下创建硬件输入设备节点(这里的硬件设备就是我们的触摸屏了)。当手指触动触摸屏时，硬件设备通过设备节点像内核(其实是InputManager管理)报告事件，`InputManager` 经过处理将此事件传给 Android系统的一个系统Service： `WindowManagerService` 。

![TouchEvent01](/album/2014-09-30-details-dispatch-onTouch-Event-in-Android-01.gif)

`WindowManagerService`调用dispatchPointer()从存放WindowState的z-order顺序列表中找到能接收当前touch事件的 WindowState，通过IWindow代理将此消息发送到IWindow服务端(IWindow.Stub子类)，这个IWindow.Stub属于ViewRoot(这个类继承Handler，主要用于连接PhoneWindow和WindowManagerService)，所以事件就传到了ViewRoot.dispatchPointer()中.

我们来看一下ViewRoot的dispatchPointer方法：

{%highlight java linenos%}
    public void dispatchPointer(MotionEvent event, long eventTime,
            boolean callWhenDone) {
        Message msg = obtainMessage(DISPATCH_POINTER);
        msg.obj = event;
        msg.arg1 = callWhenDone ? 1 : 0;
        sendMessageAtTime(msg, eventTime);
    }
{%endhighlight%}

dispatchPointer方法就是把这个事件封装成Message发送出去，在ViewRoot Handler的handleMessage中被处理，其调用了mView.dispatchTouchEvent方法(mView是一个PhoneWindow.DecorView对象)，PhoneWindow.DecorView继承FrameLayout(FrameLayout继承ViewGroup，ViewGroup继承自View),DecorView里的dispatchTouchEvent方法如下.
这里的Callback的cb其实就是Activity的attach()方法里的设置回调。

{%highlight java linenos%}
//in file PhoneWindow.java
public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();
            if (mEnableFaceDetection) {
                int pointCount = ev.getPointerCount();

                switch (ev.getAction() & MotionEvent.ACTION_MASK) {
                case MotionEvent.ACTION_POINTER_DOWN:
			
			.......

            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }
//in file Activity.java -> attach()
        mFragments.attachActivity(this, mContainer, null);
        mWindow = PolicyManager.makeNewWindow(this);
        mWindow.setCallback(this);//设置回调
{%endhighlight%}

也就是说，正常情形下，当前的Activity就是这里的cb，即调用了Activity的dispatchTouchEvent方法。

**下面来分析一下从Activity到各个子View的事件传递和处理过程。**

首先先分析Activity的dispatchTouchEvent方法。

{%highlight java linenos%}
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
{%endhighlight%}
onUserInteraction() 是一个空方法，开发者可以根据自己的需求覆写这个方法(这个方法在一个Touch事件的周期肯定会调用到的)。如果判断成立返回True，当前事件就不在传播下去了。
superDispatchTouchEvent(ev) 这个方法做了什么呢？ 
getWindow().superDispatchTouchEvent(ev) 也就是调用了 `PhoneWindow.superDispatchTouchEvent` 方法，而这个方法返回的是 mDecor.superDispatchTouchEvent(event)，在内部类 DecorView(上文中的mDecor) 的superDispatchTouchEvent 中调用super.dispatchTouchEvent(event)，而DecorView继承自ViewGroup(通过FrameLayout，FrameLayout没有dispatchTouchEvent)，最终调用的是ViewGroup的dispatchTouchEvent方法。

小结一下。Event事件是首先到了 PhoneWindow 的 DecorView 的 dispatchTouchEvent 方法，此方法通过 CallBack 调用了 Activity 的 dispatchTouchEvent 方法，在 Activity 这里，我们可以重写 Activity 的dispatchTouchEvent 方法阻断 touch事件的传播。接着在Activity里的dispatchTouchEvent 方法里，事件又再次传递到DecorView，DecorView通过调用父类(ViewGroup)的dispatchTouchEvent 将事件传给父类处理，也就是我们下面要分析的方法，这才进入网上大部分文章讲解的touch事件传递流程。

为什么要从 PhoneWindow.DecorView 中 传到 Activity，然后在传回 PhoneWindow.DecorView 中呢？ **主要是为了方便在Activity中通过控制dispatchTouchEvent 来控制当前Activity 事件的分发， 下一篇关于数据埋点文章就应用了这个机制**

OK，我们要重点分析的就是ViewGroup中的dispatchTouchEvent方法。

{%highlight java linenos%}
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
			//......

            // Check for interception.
            final boolean intercepted;//是否被拦截
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {//Touch按下事件
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);//判断消息是否需要被viewGroup拦截，这个方法我们可以覆写，
                    					    //覆写生效的前提是 disallowIntercept 为FALSE，否则写了也没用
                    ev.setAction(action); // restore action in case it was changed
                } else {//不允许拦截
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
		//这个操作不是一开始down事件，我们把它置为TRUE，拦截之
                intercepted = true;
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

				//.........

                    final int childrenCount = mChildrenCount;//ViewGroup中子View的个数
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);//获取坐标，用来比对
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final View[] children = mChildren;//获取viewgroup所有的子view

                        final boolean customOrder = isChildrenDrawingOrderEnabled();//子View的绘制顺序
						////从高到低遍历所有子View，找到能处理touch事件的child View
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder ?
                                    getChildDrawingOrder(childrenCount, i) : i;//根据Order获取子view
                            final View child = children[childIndex];
							//判断是不是我们需要的View
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);//从链表里找子view
                            if (newTouchTarget != null) {//找到子view
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
				//已经找到，循环结束，目标就是newTouchTarget
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

					//.......
                        }
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
	/*dispatchTransformedTouchEvent方法中，如果child是null，那么就调用super.dispatchTouchEvent，
	*也就是ViewGroup的父类View的dispatchTouchEvent（如果我们在前面拦截了touch事件，那么就会这样处理），
	*如果不是null，则调用child.dispatchTouchEvent。
	**/
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
			//....
            }
		//........
        return handled;
    }
 /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
 
        //……
 
        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            //……
            handled = child.dispatchTouchEvent(transformedEvent);
        }
 
        // Done.
        //……
        return handled;
    }
{%endhighlight%}

我们来总结一下 ViewGroup 的 dispatchTouchEvent 的调用过程。

1. 首先判断此 MotionEvent 能否被拦截，如果是的话，能调用我们覆写 onInterceptTouchEvent来处理拦截到的事件；如果此方法返回TRUE，表示需要拦截，那么事件到此为止，就不会传递到子View中去。这里要注意，onInterceptTouchEvent 方法默认是返回FALSE。
2. 若没有拦截此Event，首先找到此ViewGroup中所有的子View，通过方法 canViewReceivePointerEvents和isTransformedTouchPointInView，对每个子View通过坐标(Event事件坐标和子View坐标比对)计算，找到坐标匹配的View。
3. 调用dispatchTransformedTouchEvent方法，处理Event事件。
{%highlight java linenos%}
//ViewGroup.java dispatchTransformedTouchEvent方法截取
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {//Event事件被截获，调用父类View的dispatchTouchEvent方法
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);//调用子View的dispatchTouchEvent方法
            }
            event.setAction(oldAction);
            return handled;
        }
{%endhighlight%}
4. 假设这个子View是一个Button，会调用Button.dispatchTouchEvent 方法，Button和它的父类TextView都没有dispatchTouchEvent方法，只能继续看父类View了，其实最终调用的还是View.dispatchTouchEvent 方法。
5. 我们继续分析View.dispatchTouchEvent 方法。mOnTouchListener 是OnTouchListener对象，由setOnTouchListener 方法设置；
{%highlight java linenos%}
public boolean dispatchTouchEvent(MotionEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;//View 内部类，管理一些listener
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                return true;
            }

            if (onTouchEvent(event)) {
                return true;
            }
        }

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        return false;//没有消费掉，只能返回false，让调用者来处理了。
    }
    /**
     * Register a callback to be invoked when a touch event is sent to this view.
     * @param l the touch listener to attach to this view
     */
    public void setOnTouchListener(OnTouchListener l) {
        getListenerInfo().mOnTouchListener = l;
    }
{%endhighlight%}

若当前ListenerInfo 方法初始化并且 li.mOnTouchListener 的值不为空且ENABLE掩码为Enable，那么调用mOnTouchListener(this,event)方法。`boolean onTouch(View v, MotionEvent event)` 这个方法是在View的内部接口 OnTouchListener中的，是一个空方法，需要用户自己来实现。拿一个Button来举例；我们覆写的onTouch()方法在这里被调用。
{%highlight java linenos%}
	button.setOnTouchListener(new OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
        		//实现自己的功能
                return true;
            }
        });
{%endhighlight%}
6. 若onTouch方法返回true，则表示被消费，不会继续传递下去；返回false，表示时间还没被消费，继续传递到
onTouchEvent 这个方法里。

{%highlight java linenos%}
   public boolean onTouchEvent(MotionEvent event) {
        //……
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    //……
                    //如果没有触发长按事件，手指动作是up，则执行performClick()方法
                        if (!mHasPerformedLongPress) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();
 
                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                //这里判断并去执行单击事件
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
 
                    break;
                case MotionEvent.ACTION_DOWN:
                        //……
                        //是否触发长按事件是在这里判断的，具体细节我就不贴出来了
                        checkForLongClick(0);
                        //……
                    break;
                    //……
            }
            return true;
        }
 
        return false;
    }
{%endhighlight%}
若状态不是CLICKABLE，那么会直接跳过判断执行return false，这意味着后续的touch事件不会再传递过来了。而大家注意看，只要是CLICKABLE，那么无论case哪个节点，最后都是return true，这样就保证了后续事件可以传递过来。
很明显在onTouchEvent 方法里面，主要就是判断应该执行哪个操作，是长按还是单击，然后去执行对应的方法。我们看看如果是单击，执行的方法：
{%highlight java linenos%}
public boolean performClick() {
        //……
 
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            //播放点击音效
            playSoundEffect(SoundEffectConstants.CLICK);
            //执行onClick方法
            li.mOnClickListener.onClick(this);
            return true;
        }
 
        return false;
    }
{%endhighlight%}
其实就是调用了我们OnClickListener里面的onClick方法。所以说，当onTouch() 和 onClick()都存在时候，肯定是先执行onTouch，之后再执行onClick；如果onTouch 把事件截获直接return true，那么 onClick 方法就不会执行了。
到这里，整个touch事件的传递过程我们就分析完了。

[**Touch事件一般调用过程总结**]()

用户点击屏幕产生Touch(包括DOWN、UP、MOVE，本文分析的是DOWN)事件 -> InputManager -> WindowManagerService.dispatchPointer() -> IWindow.Stub -> ViewRoot.dispatchPointer() -> PhoneWindow.DecorView.dispatchTouchEvent() -> Activity.dispatchTouchEvent() -> PhoneWindow.superDispatchTouchEvent -> PhoneWindow.DecorView.superDispatchTouchEvent -> ViewGroup.dispatchTouchEvent() -> ViewGroup.dispatchTransformedTouchEvent() -> 子View.dispatchTouchEvent() -> 子View.onTouch() -> 子View.onTouchEvent() -> 事件被消费结束。(**这个过程是由上往下传导**)

**如果事件没有被子View消费，**也就是说子View的`dispatchTouchEvent`返回false，此时事件由其父类处理(由下往上传导)，最后到达系统边界也没处理，就将此事件抛弃了。

###有用的参考

1. [Android FrameWork——Touch事件派发过程详解](http://blog.csdn.net/stonecao/article/details/6759189)
2. [android的窗口机制分析------事件处理](http://blog.csdn.net/windskier/article/details/6966264)
3. [ Android事件分发机制完全解析，带你从源码的角度彻底理解(下)](http://blog.csdn.net/guolin_blog/article/details/9153747)
