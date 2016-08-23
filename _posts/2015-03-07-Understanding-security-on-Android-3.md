---
layout: post
title: 全面理解 Android 安全机制(组件安全)
categories: [Android]
tags: [Security]
---

本片博客主要跟大家谈谈 Android 基本组件的安全问题。

每个组件都可在 AndroidManifest.xml 里通过属性 exported 被设置为私有或公有。私有或公有的默认设置取决于此组件是否被外部使用；例如某组件设置了 intent-filter 意味着该组件可以接收 intent，可以吧被其他应用访问，则默认的 exported 属性为 true(没有 filter 只能通过明确的类名来启动activity故相当于只有程序本身能启动)，反之为 false（意味着此组件只能由自身的 app (同 userid 或 root也行)启动）。需要注意的是 ContentProvider 默认为 true，毕竟是共享数据的组件。


公有组件能被任何应用程序的任何组件所访问。这是非常有必要的，如 MainActivity 通常是公有的，方便 应用启动；然而大多数组件需要对其加以限制，下面我们针对不同组件来讨论。


###Activity安全

私有组件此时Activity只能被自身app启动。（同user id或者root也能启动）


**私有 Activity **不能被其他应用启动相对安全。

**创建 Activity时：**

1、不指定 taskAffinity //task 管理 Activity。task 的名字取决于根 Activity的 affinity。默认设置中 Activity 使用包名做为 affinity。task 由 app 分配，所以一个应用的 Activity 在默认情况下属于相同 task。跨 task 启动 Activity 的 intent 有可能被其他 app 读取到。

2、不指定 lanchMode //默认 standard，建议使用默认。创建新 task 时有可能被其他应用读取 intent的内容。

3、设置 exported 属性为false

4、谨慎处理从 intent 中接收的数据，不管是否内部发送的 intent

5、敏感信息只能在应用内部操作

**使用 Activity时：**

6、开启 Activity 时不设置 FLAG_ACTIVITY_NEW_TASK 标签 //FLAG_ACTIVITY_NEW_TASK 标签用于创建新 task（被启动的 Activity 并未在栈中）。

7、开启应用内部 Activity 使用显示启动的方式

8、当 putExtra() 包含敏感信息目的应是 app 内的 Activity

9、谨慎处理返回数据，即可数据来自相同应用


**公开暴露的 Activity **组件，可以被任意应用启动:

创建 Activity：

1、设置 exported 属性为 true

2、谨慎处理接收的 intent

3、有返回数据时不应包含敏感信息

使用 Activity：

4、不应发送敏感信息

5、当收到返回数据时谨慎处理


[More information about Secure Activity](http://drops.wooyun.org/tips/3936)


### Service 安全

通常 Service 执行的操作比较敏感，如更新数据库，提供事件通知等，因此一定要确保访问 Service 的组件有一定权限(也就是给 Service 设置权限)。

1. 在 AndroidManifest.xml 里给 Service 设置权限(可自定义)。

2. 一般设置 exported 属性为 false(或没有 intent-filter)；如果需要给别的 app 访问即此属性设置为 true，最好做敏感操作的时候通过 checkCallingPermission() 方法来提供权限检测。

3. 不要轻易把 Intent 传递给公有的未知名的 Service；最好在所传递的 Intent 中提供完整类名，或者在 ServiceConnection 的 onServiceConnected(ComponentName, Ibinder) 里检验 Service 的包名。

### Content Provider 安全

Content Provider 为其他不同应用程序提供数据访问方式，需要更复杂的安全措施保护。

1. 读写权限分开。一旦应用程序来访，Content Provider 需要对权限检测，只有拥有只读/只写权限才允许建立连接，否则抛出 SecurityException。

2. 只读/只写一旦实施就适用于所有数据，如果播放器只要特定音乐数据，给播放器全部访问权限，岂不是权限很大，为了细分权限粒度，可以使用 Grant-uri-permission 机制来指定一个数据集。Content Provider 可通过属性 ```<grant-uri-permission>``` 为其内的 URI 设置临时访问权限。


### Broadcast Receiver 安全

应用通常用它来监听广播消息。

1. 广播发送方通常选择给每个发送 Broadcast Intent 授予 Android 权限；接收方不但需要符合 Intent filter 的接收条件，还要求 Broadcast Receiver 也必须具有特定权限(给发送方授予权限要一致)才能接收(双层过滤)。




[部分参考自 Android安全机制解析与应用实践](http://www.douban.com/note/273536080/)









