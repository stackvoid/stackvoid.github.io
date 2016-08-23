---
layout: post
title: Sublime简介
categories: [Sublime]
tags: [Sublime]
---


我将介绍如何安装使用Sublime Text。  文章大多数内容来自网上的总结归纳。

**下载安装**

直接去[Sublime官网](ST2)下载.

**配置**

*1. 安装Package control.*
   按快捷键Ctrl+`，打开命令输入界面（一般在下方），输入下面的命令后回车即可，重启Sublime Text2。

{% highlight python %}

import urllib2,os;pf='Package Control.sublime-package';ipp=sublime.installed_packages_path();os.makedirs(ipp) if not os.path.exists(ipp) else None;open(os.path.join(ipp,pf),'wb').write(urllib2.urlopen('http://sublime.wbond.net/'+pf.replace(' ','%20')).read()) 

{% endhighlight %}


*2. 支持中文*

Ctrl+Shift+P打开命令行模式，在里面输入Install Package即可搜索需要的Package。一般使用“ConvertToUTF8”和“GBK Encoding Support”即可正常读取和写入CJK格式的文件了。

*3. 主题*

Sublime Text2也是有主题的，比较不错的主题是Soda，使用Package Control安装（自行安装请Google）。按Ctrl+Shift+P打开命令窗口，输入install，打开包安装器，输入Theme - Soda，点击后安装完成。
安装完成后还需要配置，才能使用该主题，具体：菜单->Preferences->Global Setting-User，打开配置文件，配置文件使用的是Json格式，如果已经有别的配置了，别忘记加上逗号，大致上，就是需要加上一条配置，以使用该主题

"theme": "Soda Dark.sublime-theme"

*4. 新代码配色方案*

点击Preferences->Browse Packages打开包安装目录，找到Color Scheme - Default文件夹，把下载来的新的配色安装扔进去即可。

*5. 修改字体*

Preferences->File Setting-User，打开配置文件，看里面有没font_face配置项，有就直接改，没有就自己加上，可能的配置文件如下：

{% highlight html %}

{
    "color_scheme": "Packages/Color Scheme - Default/Black-Pearl2.tmTheme",
    "font_face": "微软雅黑",
    "font_size": 10.0
}
{% endhighlight %}

**有用的链接**

[将Sublime Text 2搭建成一个好用的IDE](http://www.cnblogs.com/dolphin0520/archive/2013/04/29/3046237.html)

[Zen Coding – 超快地写网页代码](http://www.appinn.com/zen-coding/)

[Sublime Text 2 实用快捷键[Mac OS X]](http://lucifr.com/2011/09/10/sublime-text-2-useful-shortcuts/)

[Sublime Text 2 右键菜单中的实用选项](http://lucifr.com/2012/02/08/useful-entries-in-sublime-text-2-context-menu/)


[vim]: http://coolshell.cn/articles/5426.html

[ST]: https://leanpub.com/sublime-productivity

[ST2]: http://www.sublimetext.com/
