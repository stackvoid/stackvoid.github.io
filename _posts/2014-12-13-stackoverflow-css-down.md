---
layout: post
title: Stackoverflow 打开慢或样式(CSS)无法正常显示问题
categories: [Tips]
tags: [Website]
---

最近使用 stackoverflow 超级不爽，加载慢不说 样式还刷不出来，极大影响工作效率。

![stackoverflow_down-01](/album/2014-12-13-stackoverflow-not-OK.png)

查了下原因，原来是stackoverflow网站指向 CloudFlare 这个牛逼的 CDN，但是国内此 CDN 被墙了，so...

该怎么解决呢？网上搜到一个以备CDN故障时快速切换的指向资源，网址是 www.sstatic.net.

![stackoverflow_down-02](/album/2014-12-13-stackoverflow-not-OK-02.png)

现在有两种办法可以解决当前的问题：

**直接点击 上图中 [Stack Exchange family of websites](http://sstatic.net/)，会跳转到[stackexchange.com](http://stackexchange.com/sites)这个网站，跳转过来后，再次刷新你要访问的网站，应该可以正常访问了。**

![stackoverflow_down-03](/album/2014-12-13-stackoverflow-not-OK-03.png)

**一劳永逸几个月的办法：手动修改本地 Host 文件**

	首先本地获取到 cdn.sstatic.net 的 IP 

> ping cdn.sstatic.net
 
    获取到 IP 后，修改 Host 文件

> Linux:

> /etc/hosts

> WINDOWS:

> C:\Windows\System32\drivers\etc\hosts
	
	将你 ping 到的 IP 地址(x.x.x.x)加到 hosts 文件中即可。
	
> x.x.x.x     cdn.sstatic.net

  第二种方法劣势就是 IP 地址有些时候不稳定，几个月后需要换一次。
  
  **推荐使用第一种办法**，毕竟类似问题很久才遇到一次。