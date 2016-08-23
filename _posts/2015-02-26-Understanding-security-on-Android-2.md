---
layout: post
title: 全面理解 Android 安全机制(Android Permission)
categories: [Android]
tags: [Security]
---

## Android App Sandbox

在上一篇博客，我们知道 Linux 中的 Sandbox 主要做隔离工作，将不同任务或用户间的耦合降到最低。Android 应用也借用了 Linux Sandbox技术，将不同 APP 之间做了隔离；APP 之间的隔离主要是资源隔离和权限访问隔离。

	<div style="max-width:600px;margin:0 auto;">
      <div style="overflow:hidden;width:100%;display:block;">
        <a href="/album/2015/2015-02-11-Understanding-security-on-Android-1.jpg">
        <img style="display:block;float:left;max-width:300px;width:50%;position:relative;" src="/album/2015/2015-02-11-Understanding-security-on-Android-1.jpg" alt="Two Android applications, each on its own basic sandbox or process."/></a>
      </div>
    </div>

如上图所示，每个 Android APP 都运行在他们自己的 Linux 线程中（UID不同），每个应用程序彼此独立，默认情况下无法访问其他应用程序资源。 APP 权限机制为应用程序之间的资源互访提供了可行性，APP必须申请到权限并经过用户授权后才能访问 Android 系统 API 或 其他阴功程序的服务。

![Two Android applications, running on the same process](/album/2015/2015-02-11-Understanding-security-on-Android-2.jpg)

如果两个 Android App 运行在同一个进程里(此时的 UID 是相同的)，可以共享数据和代码。

那么问题来了，**如何让两个 APP 运行在同一个进程里？**

- 首先，两个 APP 要用相同的 private key 来签名
- 然后，添加两个APP manifest.xml 文件中属性 **android:sharedUserId**，均设置为相同的值或名字(其实是设置成相同的UID)。 

## Android Permission 机制

Permissions 机制限制应用访问特定的资源，如照相机、网络、外部存储、查看通话记录以及某些API。

Android 定义了很多 Permissions，详细可以[参照官方文档](http://developer.android.com/reference/android/Manifest.permission.html)。

若要申请某个权限，可以在 manifest.xml 中添加 ```<user-permission>``` 属性。

```<uses-permission android:name="string" />```

android:name 代表的就是权限名称。

###自定义 Permission

APP 可以自己定义 Permission 来限制别的APP访问自己的资源。别的应用想访问此 APP 的资源，必须在自己的 AndroidManifest.xml 中添加此 Permission。自定义权限也是在 AndroidManifest.xml 通过 ```<permission>``` 标签定义。

下面我们来举例说明。

{%highlight xml%}
<manifest xmlns:android="htttp://schemas.android.com/qpk/res/android"
package="com.stackvoid.xxx"
>
<permission
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:name="com.stackvoid.android.ACCESS_PRIVATE_DATA_TEST"
        android:description="@string/permission_description"
        android:label="@string/permission_label"
        android:protectionLevel="dangerous"
		android：permissionGroup="android.permission-group.PERSONAL_INFO"
    >
</permission>
</manifest>
{%endhighlight%}

自定义一个 Permission，至少需要的元素是 name, description, label 和 protectionLevel。 

以上 XML 语句定义了一个属于 com.stackvoid.xxx 包的新权限 - ACCESS_PRIVATE_DATA_TEST，这就意味着用户如果授予另外一个程序 ACCESS_PRIVATE_DATA_TEST 权限，所有通过 xxx 应用产生的数据都可以被那个应用程序读取到。这里，我们将 protectionLevel 设置为 dangerous，PERSONAL_INFO 定义在 android.permission-group 包中，表示 ACCESS_PRIVATE_DATA_TEST 属于 PERSONAL_INFO 这一组。

这里我们最关心的选项是 protectionLevel。

- normal，默认值。系统自动授予此 Permission，在 APP 安装的时候能看到申请此 Permission。

- signature，具有相同的 Signature的 APP，才能申请此 Permission，否则，系统拒绝。

- dangerous，一般来说系统不会自动授予此 Permission，因为此 Permission 会有潜在的威胁；一般来说在使用 APP 的过程中，需要用到此权限，会弹出窗口，让用户来授权。


当自定义权限时，**一定要遵循最小权限原则**，在这个例子中，所定义的权限仅仅是外部应用可以读取到数据，但是写数据，必须定义另外一个权限。

### 实例


首先创建了两个 APP，APP A ，APP B ； APP A 自定义权限，并且定义一个静态广播；APP B 使用此权限来发送广播给 A。 

APP A 的 AndroidManifest.xml 文件如下：

{%highlight xml%}

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.stackvoid.testbutton"
    android:versionCode="1"
    android:versionName="1.0" >

    <uses-sdk
        android:minSdkVersion="7"
        android:targetSdkVersion="19" />
    <!-- 声明权限 -->
<permission
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:name="com.stackvoid.android.ACCESS_PRIVATE_DATA_TEST"
        android:description="@string/permission_description"
        android:label="@string/permission_label"
		android：permissionGroup="android.permission-group.PERSONAL_INFO"
    >
</permission>

    <application
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
        <activity
            android:name=".MainActivity"
            launcheMode="singleTask"
            android:theme="@style/android:style/Theme.NoTitleBar.Fullscreen" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <!-- 注册 Broadcast Receiver，并指定了给当前 Receiver 发送消息方需要的权限 -->
        <receiver
            android:name="com.stackvoid.testbutton.TestButtonReceiver"
            android:permission="com.stackvoid.android.ACCESS_PRIVATE_DATA_TEST" >
            <intent-filter>
                <action android:name="com.stackvoid.test.action" />
            </intent-filter>
        </receiver>
    </application>

</manifest>


{%endhighlight%}



APP B 的 AndroidManifest.xml 文件如下：

{%highlight xml%}

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.stackvoid.testsender"
    android:versionCode="1"
    android:versionName="1.0" >

    <uses-sdk
        android:minSdkVersion="7"
        android:targetSdkVersion="19" />
    <!-- 声明使用指定的权限 -->
    <uses-permission android:name="com.stackvoid.android.ACCESS_PRIVATE_DATA_TEST" />

    <application
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
        <activity
            android:name=".MainActivity"
            android:label="@string/title_activity_main" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>


{%endhighlight%}



如上 Demo， APP B 给 APP A 发送消息，因为 APP B 中使用了 A 的权限，A 可以收到了；若未在 APP B 的 AndroidManifest.xml 文件中声明使用 A 中自定义的权限，APP B发送的消息，A是收不到的，因为 A 中的广播需要相应的权限。

另外，也可在 APP B 的 AndroidManifest.xml 文件中声明权限时，添加android:protectionLevel=“signature”,指定 APP B只能接收到使用同一证书签名的 APP 发送的消息。 


###更多资料

[stackoverflow 对自定义权限的讨论](http://stackoverflow.com/questions/8816623/how-to-use-custom-permissions-in-android)

[Android 官方文档对自定义权限的说明](http://developer.android.com/training/articles/security-tips.html)
