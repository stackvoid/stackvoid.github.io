---
layout: post
title: 使用 Git 生成标准 Patch
categories: [Tips]
tags: [Website]
---

发布给别的厂商的代码，随着 Bug 的解决，需要即使通知厂商，这时候 Patch 就大显身手了。

今天跟大家分享总结如何快速生成一个标准的Git可以识别的Patch。

**1.初始化git环境**

使用
 
> git init 

> git add *  //添加所有文件到本地

> git commit -m "init"  //提交所有文件到本地

命令来初始化当前的git目录。如果当前目录有 .git 存在，可以跳过此步骤。

**2.修改你的Code**

修改完后，可以用 

> git diff

来查看你修改了哪些code，以保证正确性。

**3.提交到本地**

	git commit --amend  提交对应修改，在这里可以添加修改的说明
	
	git commit -asm   填写patch的名字
	
**4.生成patch**

使用

> git format-patch -1 

生成patch。

最后的patch名字类似 000*-**.patch

**使用patch**

拿到 patch 的消费者直接到对应目录下，用命令  

> git am *.patch  

即可打上 patch。


