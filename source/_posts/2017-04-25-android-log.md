---
title: Android Log
date: 2017-01425 01:37:06
toc: true
tags: Android
---

Android中的Log每天都接触，这篇文章旨在揭晓平时未注意到的细节


# 一、Log level的问题


一般我们使用这几个方法打Log，Log.v() Log.d() Log.i() Log.w() and Log.e()。之所以系统分出不同的打日志方法，是为了方便开发人员分析日志，那么各个日志方法有什么区别呢？以下level值，列出几个打日志方法对应的level值，


```Java

public static final int VERBOSE = 2;

public static final int DEBUG = 3;

public static final int INFO = 4;

public static final int WARN = 5;

public static final int ERROR = 6;

public static final int ASSERT = 7;

```

官方对这几个日志level的解释是这样


> The order in terms of verbosity, from least to most is ERROR, WARN, INFO, DEBUG, VERBOSE. 


意思是从啰嗦程度来排的话，最不啰嗦的是ERROR，最啰嗦的是VERBOSE，VERBOSE这个单词的意思本来就是罗嗦的意思。


# 二、开启关闭日志


# 三、adb


