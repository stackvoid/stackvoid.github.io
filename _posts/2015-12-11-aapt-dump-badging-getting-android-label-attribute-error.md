---
layout: post
title: aapt dump badging 出错
categories: [Android]
tags: [Principle]
---

关于 andorid:label 遇到一个极其隐蔽的bug。这个值不能设置为“@null”，不然上传某些应用市场(360或应用宝等)会出现解析AndroidManifest.xml error，本地用 aapt dump badging 命令时会出现一个error，

> ERROR getting 'android:label' attribute: attribute is not a string value

![01](/album/2015/2015-12-11-aapt-dump-badging.jpg)