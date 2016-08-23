---
layout: post
title:  Ubuntu13.04下编译open-jdk7
category: [Java]
tags: [JVM]
---


最近在读[《深入理解Java虚拟机》](http://book.douban.com/subject/6522893/)，第一章主要就是手动编译一个jdk。本文可能是史上最简编译jdk的方法。很多网上编译jdk的资料极度不靠谱。

编译环境：Ubuntu13.04 64位。

<font color='purple' style='font-size:13pt'>1. 下载openjdk7源代码</font>


下载源代码有两种方式。

**用Mercurial下载**

Mercurial是一个轻量级分布式版本管理工具（类似git）。 [安装方法。]( http://blog.csdn.net/tony1130/article/details/3739695 )

然后按照以下命令下载源代码。

hg clone http://hg.openjdk.java.net/jdk7u/jdk7u6 YourOpenJDK 
cd YourOpenJDK 
sh ./get_source.sh

**直接到官方下载**

最好用[官方](http://openjdk.java.net/)最新的包,以避免各种之前出现的Bug。



<font color='purple' style='font-size:13pt'>2. 下载编译所需要的包</font>

sudo aptitude build-dep openjdk-7

sudo aptitude install openjdk-7-jdk

没有aptitude命令，自行安装。


<font color='purple' style='font-size:13pt'>3. 编译</font>

进入openjdk的目录，用命令

make sanity 检查是否在build前准备好环境。

**编译脚本**

建立sh脚本，注意脚本必须拥有可执行权限。编译脚本如下：

<font color='green' style='font-size:13pt'>
export LANG=C

export ANT_HOME=/usr/share/ant

export ALT_BOOTDIR=/usr/lib/jvm/java-7-openjdk-amd64

export DEBUG_NAME=debug

export ALT_JDK_IMPORT_PATH=/usr/lib/jvm/java-7-openjdk-amd64

unset JAVA_HOME

unset CLASS_PATH

make ARCH_DATA_MODEL=64 BUILD_JAXWS=false BUILD_JAXP=false
</font>

<font color='purple' style='font-size:13pt'>4. 编译成功</font>

经过漫长的编译，成功后会出现如下信息。

![编译成功picture](http://stackvoid.qiniudn.com/2014-05-062014-06-05-build-jdk-1.png)

在build目录下，有一个编译好的jdk。

![jdk](http://stackvoid.qiniudn.com/2014-05-062014-06-05-build-jdk-2.png)


<font color='purple' style='font-size:13pt'>5. 一些有用的资料</font>

1. [Eclipse调试Hotspot](http://blog.csdn.net/hengyunabc/article/details/16912775)

1. [调试Hotspot](https://github.com/codefollower/OpenJDK-Research/blob/master/hotspot/my-docs/%E6%9E%84%E5%BB%BA%E4%B8%8E%E8%B0%83%E8%AF%95-Linux.md)



<font color='blue' style='font-size:14pt'>真爱生命，远离百度。</font>

(完)
