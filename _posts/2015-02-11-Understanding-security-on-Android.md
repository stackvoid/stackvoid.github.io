---
layout: post
title: 全面理解 Android 安全机制(Linux基础)
categories: [Android]
tags: [Security]
---

<Br />
## <font color="green">Linux Sandbox 机制</font>
--------------------

Linux Sandbox(沙箱)机制，主要用于程序或进程隔离，限制不可信程序或进程的权限，将程序或进程造成的不良后果降到最低。Sandbox 机制可以为不可信的程序提供虚拟化的内存、网络、外部存储等资源；而对于用户来说是透明的。

####Sandbox在进程中的应用

- 每个进程都有自己独立的地址空间，MMU 硬件强制分离地址空间，每个进程都有自己的权限边界
- 内核在系统中有强制的接口层(Kernel安全策略)
- root用户拥有至高权限，其他用户拥有自主访问 DAC(Discretionary Access Control Model) 访问权限。

####Linux 用户

Linux 支持多用户工作，每个用户之间都使用了沙箱达到相互隔离，互不影响。

<Br />
## <font color="green">Linux 权限机制</font>
-----------------

主要讨论文件权限、UID、GID 和 Capability。

####文件权限

文件权限有读、写和执行权限，分别对应 r w x 这几个字母。

如下一个例子。

-rwxr-xr-x  1 stackvoid_ming bt 23902 Jul  2  2014 repo

第一个 “-” 代表文件类型，默认为普通文件，若为 “d” 则代表目录(dir)文件， “l” 代表链接(Link)文件。

第2位到第4位（即 rwx）代表的是此文件拥有者的权限，文件拥有者有读写执行权限。
第5位到第7位（即 r-x）代表的是 GROUP（用户组）对这个文件的权限，这里没有写权限，有读和执行权限。
第8位到第10位（即 r-x）代表的是 Other（别的用户，非文件拥有者和非用户组用户），这里有读和执行的权限。

####UID/GID

**UID:** User Identifier；全称用户标识符，在类UNIX系统中是内核用来辨识用户的一个无符号整型数值，亦是UNIX文件系统与进程的必要组成部分之一。


**有效用户ID**（Effective UID，即EUID）与有效用户组ID（Effective Group ID，即EGID）在创建与访问文件的时候发挥作用；具体来说，创建文件时，系统内核将根据创建文件的进程的EUID与EGID设定文件的所有者/组属性，而在访问文件时，内核亦根据访问进程的EUID与EGID决定其能否访问文件。

**真实用户ID**（Real UID,即RUID）与真实用户组ID（Real GID，即RGID）用于辨识进程的真正所有者，且会影响到进程发送信号的权限。没有超级用户权限的进程仅在其RUID与目标进程的RUID相匹配时才能向目标进程发送信号，例如在父子进程间，子进程从父进程处继承了认证信息，使得父子进程间可以互相发送信号。

**暂存用户ID**（Saved UID，即SUID）于以提升权限运行的进程暂时需要做一些不需特权的操作时使用，这种情况下进程会暂时将自己的有效用户ID从特权用户（常为root）对应的UID变为某个非特权用户对应的UID，而后将原有的特权用户UID复制为SUID暂存；之后当进程完成不需特权的操作后，进程使用SUID的值重置EUID以重新获得特权。在这里需要说明的是，无特权进程的EUID值只能设为与RUID、SUID与EUID（也即不改变）之一相同的值。

**文件系统用户ID**（File System UID，即FSUID）在Linux中使用，且只用于对文件系统的访问权限控制，在没有明确设定的情况下与EUID相同（若FSUID为root的UID，则SUID、RUID与EUID必至少有一亦为root的UID），且EUID改变也会影响到FSUID。设立FSUID是为了允许程序（如NFS服务器）在不需获取向给定UID账户发送信号的情况下以给定UID的权限来限定自己的文件系统权限。

**GID**(Group identifier)跟 UID 一样，也是无符号整型数值。对某个文件非拥有者，如果有组权限，可以按照组权限来访问此文件，一般来说组权限要高于 OTHER，低于文件拥有者权限。

例如上面的例子：

-rwxr-xr-x  1 stackvoid_ming bt 23902 Jul  2  2014 repo

stackvoid_ming 指的就是 user id，bt 指的是 Group ID。

####Capability

Linux系统分为root用户和非root用户，root用户至高无上，想干嘛干嘛，至于非root用户权限有限，想root权限，可以通过setuid实现。如果一个进程想要一个大于非root用户权限，只能将自己提升到root权限；实际上此进程并不需要这么多权限，这可能会导致很多风险(一个进程权限太大)，所以POSIX制定了Capability机制来细分root的权限。

目前内核(3.14，\include\uapi\linux\capability.h)有36个capability，未来根据需求会继续增加。
Capability是可配选项，可以通过CONFIG_SECURITY_CAPABILITIES选项设置(kernel2.5.27之后，3.8后默认enable)。看似很美好的权限细分，实际上大多数程序都还没使用。

程序中的Capability实际上就是一堆位图。例如：

CapInh：00000000000025C0

希望进程拥有什么权限(Capability)，设置这里对应的bit位即可。我们可以查看/proc/<pid>/status来查看进程的capability。Capability 不是我们重点关心的，这里点到即可，详细可以参考[这篇文档](http://man7.org/linux/man-pages/man7/capabilities.7.html)。



##Reference
------------

[wikipedia-UID](http://en.wikipedia.org/wiki/UID)
[Linux-capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)