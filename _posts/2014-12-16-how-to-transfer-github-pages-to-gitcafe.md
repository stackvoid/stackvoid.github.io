---
layout: post
title: 如何备份Github博客至GitCafe
categories: [Tips]
tags: [Website]
---

开通博客半年多了，一直将博客托管到 Github 上，使用 Github Pages 的免费服务；最近发现一个令人不安的事实，我的博客在其他省份解析的很慢，通过 CNZZ 和 Google Analytics 发现有些时候博客打开速度慢出翔，当时就想买独立 VPS 了，“受够了”这种免费服务了；直到一朋友跟我说，就是用 Linode 国内访问有些时候延迟也比较厉害，彻底崩溃，直到在微博上看到有人分享 GitCafe！这速度果然刁刁的！

![01](/album/2014-12-16-1.jpg)

本文详细记录了我在备份 Github 博客到 Gitcafe 的点滴。

**1.注册GitCafe账号**

点击[GitCafe](http://gitcafe.com/signup?invited_by=stackvoid)，填上各种信息注册即可(**注：这是邀请链接，不喜欢被邀请的可以把邀请人信息删除掉，^_^**)

![02](/album/2014-12-16-1.5.png)

然后在 GitCafe 上创建一个项目，注意 项目名字必须跟用户名相同。

![03](/album/2014-12-16-1.6.jpg)

**2.克隆 GitCafe的项目到本地**

首先将 SSH 信息配置到 GitCafe上（与配置 GitHub 一样），然后将新建的项目克隆到本地(与服务器建立好联系)。

![03](/album/2014-12-16-2.jpg)

**3.提交博客源码到 GitCafe**

首先将 GitHub Pages 所有的文件拷贝到刚从GitCafe克隆下来的目录中(我的目录是 stackvoid)。

然后提交所有的文件。并push到服务器上。

![04](/album/2014-12-16-4.jpg)

然后在本地建立 gitcafe-pages 分支，并同步到服务器上，GitCafe 的 Pages 服务仅通过 gitcafe-pages 分支来解析。

![04.1](/album/2014-12-16-5.png)

把 gitcafe-pages push 到服务器上，快点试试你的博客能不能打开吧！访问地址是：用户名.gitcafe.com。例如我博客在 Gitcafe 的地址是 stackvoid.gitcafe.com 。

**4.DNS设置**

如果你有私有网址(比如我的 [stackvoid.com](http://stackvoid.com)，可以通过设置 DNS 让国内访问走 GitCafe，国外访问走 GitHub。我用的是 DNSPOD，如下图设置就好。设置完后要让子弹飞一大会才能生效(DNS生效，你懂的...)。

![05](/album/2014-12-16-5-5.png)


等着享受飞一般的速度吧！！！


----------------------2014-12-17-update----------------------------------

刚高兴一天，Gitcafe 发邮件说：

> 我们将于周四凌晨零点 （12月18日 00:00） 暂时关闭自定义域名功能。 

> 我们承诺将在两周内（12月31日前）重新开放 Pages 服务的自定义域名功能。


![06](/album/2014-12-16-6.jpg)

> 不过对用户影响不大，只要 DNSPOD 设置正确了就行。
> 私有域名直接交给 GitCafe 的用户可能会受很大影响。

