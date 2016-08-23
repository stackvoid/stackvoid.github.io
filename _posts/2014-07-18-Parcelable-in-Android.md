---
layout: post
title: Android序列化：Parcelable VS Serializable
categories: [Android]
tags: [Parcelable]
---

在Activity之间传送对象数据要比传送简单参数麻烦很多，传送简单数据，我们只要把数据放到Intent上即可，如果传送String或Integer这样的简单数据，也是可行的。

{%highlight java%}

String strinParam = "String Parameter"; 
Integer intParam = 5;   
Intent i = new Intent(this, MyActivity.class); 
i.putExtra("com.stackvoid.stringParam", stringParam); 
i.putExtra("com.stackvoid.intParam", intParam);   
startActivity(i); 

{%endhighlight%}

##序列化的作用

Serializable的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。
而Android的Parcelable的设计初衷是因为Serializable效率过慢，为了在程序内**不同组件间以及不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。**


##效率

在内存间数据传输时推荐使用Parcelable，如activity间传输数据，而Serializable可将数据持久化方便保存，所以在需要保存或网络传输数据时选择Serializable，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化。

部分Parcelable源码(Frameworks/base/core/jni/android_util_Binder.cpp)
{%highlight c++%}
static void android_os_Parcel_writeInt(JNIEnv* env, jobject clazz, jint val)
{
    Parcel* parcel = parcelForJavaObject(env, clazz);
    if (parcel != NULL) {
        const status_t err = parcel->writeInt32(val);
        if (err != NO_ERROR) {
            jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        }
    }
}

static void android_os_Parcel_writeLong(JNIEnv* env, jobject clazz, jlong val)
{
    Parcel* parcel = parcelForJavaObject(env, clazz);
    if (parcel != NULL) {
        const status_t err = parcel->writeInt64(val);
        if (err != NO_ERROR) {
            jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        }
    }
}
{%endhighlight%}

更多代码见 Frameworks/base/include/binder/parcel.h和Frameworks/base/libs/binder/parcel.cpp。

分析可知道内存数据操作Parcelable比Serializable高效原因有：

1. 整个读写全是在内存中进行，主要是通过malloc()、realloc()、memcpy()等内存操作进行，所以效率比JAVA序列化中使用外部存储器会高很多；

1. 读写时是4字节对齐的，#define PAD_SIZE(s) (((s)+3)&~3)这句宏定义就是在做这件事情；

1. 如果预分配的空间不够时newSize = ((mDataSize+len)*3)/2;会一次多分配50%；

1. 对于普通数据，使用的是mData内存地址，对于IBinder类型的数据以及FileDescriptor使用的是mObjects内存地址。后者是通过flatten_binder()和unflatten_binder()实现的，目的是反序列化时读出的对象就是原对象而不用重新new一个新对象。

##使用

对于Serializable，类只需要实现Serializable接口，并提供一个序列化版本id(serialVersionUID)即可。而Parcelable则需要实现**writeToParcel、describeContents函数以及静态的CREATOR变量，实际上就是将如何打包和解包的工作自己来定义**，而序列化的这些操作完全由底层实现。

{%highlight java%}
package parcelable_test.com;
import android.os.Parcel;  
import android.os.Parcelable; 
import android.util.Log;
public class Person implements Parcelable{
private String name;
private int age;
private static final String TAG = ParcelableTest.TAG; 
public String getName() {
return name;
}
public void setName(String name) {
this.name = name;
}
public int getAge() {
return age;
}
public void setAge(int age) {
this.age = age;
}
public static final Parcelable.Creator<Person> CREATOR = new Creator<Person>() {
@Override
public Person createFromParcel(Parcel source) {
Log.d(TAG,"createFromParcel");
Person mPerson = new Person();
mPerson.name = source.readString();
mPerson.age = source.readInt();
return mPerson;
}
@Override
public Person[] newArray(int size) {
// TODO Auto-generated method stub
return new Person[size];
}
};
@Override
public int describeContents() {
// TODO Auto-generated method stub
Log.d(TAG,"describeContents");
return 0;
}
@Override
public void writeToParcel(Parcel dest, int flags) {
// TODO Auto-generated method stub
Log.d(TAG,"writeToParcel");
dest.writeString(name);
dest.writeInt(age);
}
}

//使用

public class XXX ...{
...
public static final String KEY = "key";
Person mPerson = new Person();
mPerson.setName("tom");
mPerson.setAge(25);
Intent mIntent = new Intent(this,parcelable_test.com.ParcelableTest2.class);
Bundle mBundle = new Bundle();
mBundle.putParcelable(KEY, mPerson);
mIntent.putExtras(mBundle);
startActivity(mIntent);
...
}

//获取Parcel

Person mPerson = (Person)getIntent().getParcelableExtra(ParcelableTest.KEY);
{%endhighlight%}

在mBundle.putParcelable(KEY, mPerson);时，调用了Person类中的public void writeToParcel(Parcel dest, int flags)方法，并向dest写数据，在 Person mPerson = (Person)getIntent().getParcelableExtra(ParcelableTest.KEY);的时候，调用了Person类中的public Person createFromParcel(Parcel source) 方法，创建了一个Person对象，并给这个对象的属性赋值，这里的Parcel source和Parcel dest,是相同的，然后返回这个Person对象。最后就可以打印出mPerson的属性信息了。


##选择

1. Activity等同一个设备内存里互相传递复杂数据（对象）用Parcelable。因为Serializable序列化采用大量反射，且会产生很多临时变量，以至于移动设备高负载。

1. Parcelable只能在内存中处理，不能讲数据存储序列化到磁盘上或网络传输，尽管Serializable效率低，这种情况还得用Serializable。


###参考

1. [Android – Parcel data to pass between Activities using Parcelable classes - See more at: http://shri.blog.kraya.co.uk/2010/04/26/android-parcel-data-to-pass-between-activities-using-parcelable-classes/#sthash.dejJls13.dpuf](http://shri.blog.kraya.co.uk/2010/04/26/android-parcel-data-to-pass-between-activities-using-parcelable-classes/)

1. [PARCELABLE VS. JAVA SERIALIZATION IN ANDROID APP DEVELOPMENT](http://www.3pillarglobal.com/insights/parcelable-vs-java-serialization-in-android-app-development)

1. [Java 序列化的高级认识](http://www.ibm.com/developerworks/cn/java/j-lo-serial/index.html)

1. [探索Android中的Parcel机制（上）](http://blog.csdn.net/caowenbin/article/details/6532217)

1. [序列化：Parcelable、Serializable](http://blog.csdn.net/u011936381/article/details/19120803)
