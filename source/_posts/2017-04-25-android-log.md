---
title: Android Log
date: 2017-01425 01:37:06
toc: true
tags: Android
---

# 由TimingLogger引出的问题

前两天用Android官方推荐的一个辅助打计时日志的工具：[TimingLogger](https://developer.android.com/reference/android/util/TimingLogger.html)

这个工具确实挺方便的，可以达到以下效果


代码如下

```java
 TimingLogger timings = new TimingLogger(TAG, "methodA");
 // ... do some work A ...
 timings.addSplit("work A");
 // ... do some work B ...
 timings.addSplit("work B");
 // ... do some work C ...
 timings.addSplit("work C");
 timings.dumpToLog();
```

以上代码会打印如下日志

```java
D/TAG     ( 3459): methodA: begin
D/TAG     ( 3459): methodA:      9 ms, work A
D/TAG     ( 3459): methodA:      1 ms, work B
D/TAG     ( 3459): methodA:      6 ms, work C
D/TAG     ( 3459): methodA: end, 16 ms
```

但是发现在手头的huawei机器上打不出日志来，结果发现这个[TimingLogger](https://developer.android.com/reference/android/util/TimingLogger.html)工具判断了当前Log所支持的级别是否达到VERBOSE的级别，如果不是，则不会打印最终的结果出来。如下

```java
mDisabled = !Log.isLoggable(mTag, Log.VERBOSE);
...
...
public void dumpToLog() {
        if (mDisabled) return;
        Log.d(mTag, mLabel + ": begin");
        final long first = mSplits.get(0);
        long now = first;
        for (int i = 1; i < mSplits.size(); i++) {
            now = mSplits.get(i);
            final String splitLabel = mSplitLabels.get(i);
            final long prev = mSplits.get(i - 1);
            Log.d(mTag, mLabel + ":      " + (now - prev) + " ms, " + splitLabel);
        }
        Log.d(mTag, mLabel + ": end, " + (now - first) + " ms");
    }
```
而最终发现我的华为机器Log.isLoggable(mTag, Log.VERBOSE)返回了false，所以TimingLogger无法打印日志出来，接下来我们试试怎么修改这个日志级别


Android中的Log每天都接触，我们先来梳理一下基础知识。


# Log level


一般我们使用这几个方法打Log，Log.v() Log.d() Log.i() Log.w() and Log.e()。之所以系统分出不同的打日志方法，是为了方便分析日志，
另一个更直观的感受是IDE下会对不同级别的日志显示不同的颜色。

那么各个日志方法有什么区别呢？每个方法都对应一个level值，如下：


```Java
public static final int VERBOSE = 2;
public static final int DEBUG = 3;
public static final int INFO = 4;
public static final int WARN = 5;
public static final int ERROR = 6;
public static final int ASSERT = 7;
```

官方对这几个日志level排序的解释是这样

> The order in terms of verbosity, from least to most is ERROR, WARN, INFO, DEBUG, VERBOSE. 


意思是从啰嗦程度来排，最不啰嗦的是ERROR，最啰嗦的是VERBOSE，VERBOSE这个单词的本来就是啰嗦的意思。


# 二、开启关闭日志





# 三、adb








