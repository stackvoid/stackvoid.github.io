---
layout: post
title:  常用的Shell命令
category: Shell
tags: [Shell]
---



本文主要总结一些常用的shell命令。






### 文本处理类
============

#### 查找特定文件名


grep -rn “name” 目录

例如  grep -rn “ming” *   寻找当前目录下所有文件 有ming字符的列出。
[Linux的五个查找命令](http://www.ruanyifeng.com/blog/2009/10/5_ways_to_search_for_files_using_the_terminal.html)

#### make -p


即使data这个文件夹不存在，也可以创建他的子目录，当然同同时，他也被创建。

mkdir -p /data/coredump

#### head  tail


head 是显示一个文件的内容的前多少行；

head  -n  行数值  文件名；

比如我们显示/etc/profile的前10行内容，应该是：

[root@localhost ~]# head -n 10 /etc/profile

tail 工具，显示文件内容的最后几行，用法比较简单；

tail   -n  行数值  文件名；

比如我们显示/etc/profile的最后5行内容，应该是：

[root@localhost ~]# tail  -n 5 /etc/profile


#### 解压缩文件


{% highlight java %}

解压

tar –xvf file.tar //解压 tar包

tar -xzvf file.tar.gz //解压tar.gz

tar -xjvf file.tar.bz2   //解压 tar.bz2

tar –xZvf file.tar.Z   //解压tar.Z

unrar e file.rar //解压rar

unzip file.zip //解压zip

压缩

tar –cvf jpg.tar *.jpg //将目录里所有jpg文件打包成tar.jpg

tar –czf jpg.tar.gz *.jpg   //将目录里所有jpg文件打包成jpg.tar后，并且将其用gzip压缩，生成一个gzip压缩过的包，命名为jpg.tar.gz

tar –cjf jpg.tar.bz2 *.jpg //将目录里所有jpg文件打包成jpg.tar后，并且将其用bzip2压缩，生成一个bzip2压缩过的包，命名为jpg.tar.bz2

tar –cZf jpg.tar.Z *.jpg   //将目录里所有jpg文件打包成jpg.tar后，并且将其用compress压缩，生成一个umcompress压缩过的包，命名为jpg.tar.Z

rar a jpg.rar *.jpg //rar格式的压缩，需要先下载rar for linux

zip jpg.zip *.jpg //zip格式的压缩，需要先下载zip for linux

{% endhighlight %}

#### 字符串处理


*找文件夹下包含字符串的文件*

例：查找/usr/local目录下所有包含”rubyer.me”的文件。

> grep -lr 'rubyer.me' /usr/local/*


*vim替换单个文件中所有字符串方法*

例：替换当前文件中所有old为new


> :%s/old/new/g
\#%表示替换说有行，g表示替换一行中所有匹配点。


*替换文件夹下包含字符串的文件*

sed结合grep

例：要将目录/www下面所有文件中的zhangsan都修改成lisi，这样做：


> sed -i "s/old/new/g" `grep old -rl /www`


### 进程管理类
============

#### 查找进程


ps -ef|grep -w  indexd_admin_mcd.pid|grep -v grep|wc -l

ps -ef 查找进程    grep -v  查找不存在  grep -w 强制 PATTERN 仅匹配整个词

查找进程中为 indexd_admin_mcd.pid的进程，并且排除掉grep的进程。最后计数，这样进程的个数。

#### 杀死进程


kill -9 pid，是不顾后果的强制终止(如果的你的速度够快，有时候是和ctrl＋c是一样的)

kill -15 pid，是先关闭和其有关的程序，再将其关闭

#### linux下如何查询已知进程运行目录


 ls -l /proc/PID/cwd
 

### 监视类
===========

top命令
<br/><br/>

### 内存管理类
==========

#### ipcs  查看共享内存的使用的情况


ipcrm -M  shmid  关闭共享内存
