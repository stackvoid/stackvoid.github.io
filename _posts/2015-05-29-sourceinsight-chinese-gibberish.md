---
layout: post
title: 批处理解决 SourceInsight 中文乱码
categories: [Tips]
tags: [SourceInsight]
---

在 Windows 下把下面的代码做成批处理 .bat 文件，把源码目录拷贝到** E:\hehe **运行即可。

（注意跟进你自己的情况修改DIR目录以及修改想要转换的格式文件），下面的 code 仅仅是解决目录 E:\tmp 下的 java 和 xml 文件。

{%highlight python%}

@echo off
set DIR=E:\hehe
for /R %DIR% %%i in (*.java *.xml) do (
echo %%i
native2ascii -encoding UTF-8 %%i %DIR%\temp
native2ascii -reverse %DIR%\temp %%i
)
pause

{%endhighlight%}

[转载请注明](http://stackvoid.com/sourceinsight-chinese-gibberish/)