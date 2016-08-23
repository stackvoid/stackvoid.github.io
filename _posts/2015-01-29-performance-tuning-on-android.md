---
layout: post
title: Android App 性能优化实践
categories: [Android]
tags: [Optimization]
---

> 本文记录了Android App优化需要用到的工具和以及在实践中的Tips。也算对我这半年来部分工作的总结。


##<font color="green">工具</font>
----------------

[Hierarchy Viewer](http://developer.android.com/tools/help/hierarchy-viewer.html) 是 Android SDK 自带的 Layout 嵌套检查工具，以可视化的布局角度直观获取 Layout 布局设计和各种属性信息，来帮助我们完成优化布局的设计。需要注意的是，出于安全考虑 Hierarchy Viewer **只能连接Android开发版手机(需要安装ViewServer)或是模拟器**。

![1](/album/2015/2015-01-29-performance-tuning-on-android-1.png)

注意上图右半部分显示的时间

- Measure： 0.977ms
- Layout： 0.167ms
- Draw： 2.717ms

我们知道Android View在绘制图形的时候主要耗时的操作就在 Measure、Layout 和 Draw 这三个过程；并且任何一个 View 绘制时间不能超过 16.7ms(每秒60帧才能保证流畅度)。

如果 UI 出现卡顿或掉帧，那么 Hierarchy Viewer 这个工具及其有用，可以分析当前 View 是哪些 View 以及是 View 的哪个过程加载延迟，通过这些信息基本可定位到局部 Code。

**如何让QC快速追踪和定位性能问题？**

当然使用 Android 开发者工具里的 [**Profile GPU rendering** (GPU呈现模式分析)](http://developer.android.com/about/versions/jelly-bean.html) 工具(Android4.1以上)。它能够从屏幕上活动的所有Android Activity生成性能视图，其中绿线代表 **16ms**，频繁超过此线的 Activity 就要排查性能问题了。


**如何定位到某个方法？**

用 Hierarchy Viewer 知道是哪一个子 View 耗时比较多，找到此 View 的Code，那么如何定位到具体某个方法里呢？ 当然需要 [traceview](http://developer.android.com/tools/debugging/debugging-tracing.html) 工具。traceview 工具十分强大，可以轻松把每个方法占用 CPU 时间计算出来，找到占用时间最长的方法，然后分析此方法即可。

[**Lint**](http://developer.android.com/tools/debugging/improving-w-lint.html) 工具已经集成于 Android Studio，同样是非常强大的工具。它会给出 Layout 优化提示(既包括图片资源、layout文件，也有定义的String常量和Color常量以及Layout写法不规范)，告诉你哪些资源没有被引用，Manifest文件的错误等；我主要用 lint 来哪些资源文件没有被引用到(给APK瘦身)，以及部分代码不规范的地方。

####内存优化工具

Memory Monitor：查看整个app所占用的内存，以及发生GC的时刻，短时间内发生大量的GC操作是一个危险的信号(发生内存抖动)。

Allocation Tracker：追踪内存的分配。

Heap Tool：查看当前内存快照，便于对比分析哪些对象有可能是泄漏了的。

##<font color="green">布局优化</font>
----------
####布局标签

**```<include>``` 标签**，将布局中公共部分提取出来共用；例如网易新闻一条新闻的标题栏和评论界面的标题栏。

**```<viewstub>``` 标签**，同 include，可引入布局，但是默认情况引入的布局不会占用资源，在解析当前 Layout 时节省计算、内存资源。当需要加载此 View 的时候，需要动态 inflate 起来。

> Tips：将一个view设置为GONE不会被系统解析，从而提高layout解析速度，而VISIBLE和INVISIBLE这两个可见性属性会被正常解析。

[**```<merge>``` 标签**](http://developer.android.com/training/improving-layouts/reusing-layouts.html)，解决 Layout 嵌套过多的问题，通过工具通过 hierarchy viewer 可直观的显示出来。

#### 其他

减少 inflate 次数：inflate 是比较耗资源的，当内存够用时，可以将 View 缓存起来，下次直接使用；用空间换时间。

ListView 优化，[请见我另外一篇博客](http://stackvoid.com/list-view-optimization-best-practice-android/)。

关于 Layout 优化，推荐一篇博客，给我很大帮助，[性能优化系列](http://www.trinea.cn/android/performance/)。

##<font color="green">代码Tips</font>
-----------

[性能优化之Java(Android)代码优化](http://www.trinea.cn/android/java-android-performance/)，这篇博客详细介绍了如何进行代码优化，包括缓存、数据存储、异步、数据库和网络等操作的优化。

关于缓存，上文没有提到一个重要的库：DiskLruCache；DiskLruCache 是关于数据硬盘缓存的，[Android DiskLruCache完全解析，硬盘缓存的最佳方案](http://blog.csdn.net/guolin_blog/article/details/28863651) 这篇博客详细介绍了 DiskLruCache 使用方法和注意事项。

**避免随意使用静态变量**，当某个对象被定义为stataic变量所引用，虚拟机通常是不会回收这个对象所占有的内存。

**避免过多过常的创建java对象**，JVM 创建和回收耗时，频繁使用对象，最好创建缓存；每次回收对象，都是 STW(Stop the World)，所以如果对象过多，可能引起卡顿(大于16ms，引起掉帧)。可用 Memory Monitor 或 Allocation Tracker 工具来查看这类问题。

**多使用局部变量**，函数执行完，就释放内存被虚拟机回收。

**使用StringBuilder和StringBuffer进行字符串连接**，尤其在做 SQL 拼装的时候。

**单线程应尽量使用HashMap, ArrayList**，如果不确定是单线程还是多线程，建议还是用 ConcurrentHashMap...

**尽量在finally块中释放资源**，例如很多 Cursor。

**慎用异常**，创建一个异常时，需收集一个栈记录(stack track)，用于描述异常是在何处创建的。构建这些此栈时需要为运行时栈做一份快照，这一部分开销很大。

##<font color="green">View绘制</font>
-------------
####过度绘制问题

为什么会出现过度绘制：多个 View 重叠，复杂 Layout 叠加；导致 GPU 需要绘制多层，有些时候非常耗时。

[Android性能优化之过渡绘制](http://www.androidperformance.com/android-performance-optimization-overdraw-1.html)，这篇博客作者用实例来解决过度绘制的问题，解决过度绘制问题时，作者也使用了我们上面介绍的几个工具。

####<font color="#611774">View局部更新</font>

一些复杂的 View，如果每次 View 有局部更新都要重新绘制 View的话，GPU 会显得力不从心。通过[canvas.clipRect() 方法](http://developer.android.com/reference/android/graphics/Canvas)来让系统识别可绘制区域。这个方法可以指定一块矩形区域，只有在这个区域内才会被绘制，其他的区域会被忽视。clipRect方法节约了CPU与GPU资源，不会绘制clipRect区域外的地方，仅仅绘制内容在矩形区域内的组件。

##电量优化
------------
尽量减少唤醒屏幕的次数与持续的时间(屏幕是用电大户)，用WakeLock来处理唤醒的问题，能够正确执行唤醒操作并根据设定及时关闭操作进入睡眠状态，使用 wakelock.acquice() 方法，一定要加上超时处理(例如释放锁)。

等到设备处于充电状态或者电量充足的时候才进行耗时耗电操作(如分享传送数据、图片处理等)

触发网络请求的操作，每次都会保持无线信号持续一段时间，我们可以把零散的网络请求打包进行一次操作，避免过多的无线信号引起的电量消耗(例如APP的数据采集)。

**Battery Historian Tool**(Android 5.0)这个工具可以详细查看各类应用的用电情况。

## APK 瘦身
-------------

#### 代码瘦身

库的使用可以极大方便开发者快速开发产品，但也引入了潜在的 bug 以及库过大导致APK过大的问题。移除没有用的 dependency libraries 是一个很好的建议。另外适当的给库瘦身(提取自己想要的功能)也很重要。如果对 APK 代码非常熟悉可以使用 Proguard （会遍历你的所有代码然后找出无用处的代码）优化。
 
#### 控制资源文件

剔除没有用的资源文件(使用 Lint 可轻松检测到)。

资源里的照片先进行压缩再使用。合适的时候可以用代码控制图片大小作为不同分辨率屏幕的资源。

为应用提供 hdpi, xhdpi 和 xxhdpi 这几个屏幕密度的支持。如果某些设备不是这几个屏幕密度的，不用担心，Android 系统会自动使用存在的资源为设备计算然后提供资源文件。



## 总结
--------
出现卡顿的根本原因：系统绘制 View 超过 **16ms**，出现掉帧才导致卡顿或不流畅。解决方法：

-  Hierarchy Viewer，Profile GPU rendering，traceview
-  抽象布局标签，使用标签 include、viewstub、merge
-  多使用缓存
-  尽量避免过度绘制
-  自定义复杂 View，动态更新 View 内容
-  正确使用 wakelock，保持 App 用电量




### Reference
-------------
[Android Performance Patterns](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)

[Android性能优化之过渡绘制](http://www.androidperformance.com/android-performance-optimization-overdraw-2.html)

[Android性能优化典范 ](http://hukai.me/android-performance-patterns/)

[Performance Tuning On Android ](http://blog.venmo.com/hf2t3h4x98p5e13z82pl8j66ngcmry/performance-tuning-on-android)

