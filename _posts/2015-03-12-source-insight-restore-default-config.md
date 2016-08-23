---
layout: post
title: 恢复 Source Insight 默认配置
categories: [Tips]
tags: [Source Code]
---

近期在重温 Volley 的源码，手贱乱点击 Source Insight 这个阅读源码神器，导致 Source Insight 窗口完全乱掉，只有 source code，源码的索引，以及源码文件都不见了。

![01](/album/2015/2015-03-12-source-insight-restore-default-config-1.jpg)

有两种方法恢复到之前的状态：

1. Ctrl + O 可以重新打开项目代码的窗口。

2. 先关闭 Source Insight 软件，然后找到下图的文件夹中的文件，删除这两个文件，再次启动软件即可。

![02](/album/2015/2015-03-12-source-insight-restore-default-config-2.jpg)



因为 Source Insight 主要就是方便看源码，建议使用第二种方法彻底恢复到初始配置。

**那么问题来了，字体应该怎么设置的大一点？**

1. Alt + T，打开配置窗口。

2. 点击 Screen Font 选项，配置即可。
