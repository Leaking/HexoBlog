---
title: huawei上日志输出的问题
date: 2017-04-25 01:37:06
toc: true
tags: Android
---

前两天用Android官方推荐的一个辅助打计时日志的工具：[TimingLogger](https://developer.android.com/reference/android/util/TimingLogger.html).这个工具确实挺方便的，可以达到以下效果


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
而最终发现我的华为机器`Log.isLoggable(mTag, Log.VERBOSE)`返回了false，所以`TimingLogger`无法打印日志出来，接下来我们试试怎么修改这个日志级别.


Android中的Log每天都接触，我们先来梳理一下基础知识。



一般我们使用这几个方法打Log，Log.v() Log.d() Log.i() Log.w() Log.e()。之所以系统分出不同的打日志方法，是为了方便分析日志，另一个更直观的感受是IDE下会对不同级别的日志显示不同的颜色。

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


意思是从啰嗦程度来排，最不啰嗦的是ERROR，最啰嗦的是`VERBOSE`，`VERBOSE`这个单词的本来就是啰嗦的意思。


接下来在手头的hauwei mate9上运行以下代码，

```java
Log.v(TAG, "VERBOSE");
Log.d(TAG, "DEBUG");
Log.i(TAG, "INFO");
Log.w(TAG, "WARN");
Log.e(TAG, "ERROR");


Log.i(TAG, "VERBOSE isLoggable " + Log.isLoggable(TAG, Log.VERBOSE));
Log.i(TAG, "DEBUG isLoggable " + Log.isLoggable(TAG, Log.DEBUG));
Log.i(TAG, "INFO isLoggable " + Log.isLoggable(TAG, Log.INFO));
Log.i(TAG, "WARN isLoggable " + Log.isLoggable(TAG, Log.WARN));
Log.i(TAG, "ERROR isLoggable " + Log.isLoggable(TAG, Log.ERROR));
```

最终输出日志如下

```java
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: INFO
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs W/MainActivity: WARN
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs E/MainActivity: ERROR

05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: VERBOSE isLoggable false
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: DEBUG isLoggable false
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: INFO isLoggable true
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: WARN isLoggable true
05-03 22:55:18.163 10825-10825/com.example.quinn.apilabs I/MainActivity: ERROR isLoggable true
```

我们分析一下以上结果，不难得知，`VERBOSE`和`DEBUG`是打印不出来的，而`Log.isLoggable(TAG, Log.VERBOSE)`和`Log.isLoggable(TAG, Log.DEBUG)`都是返回`false`。只要某个`level`的`isLoggable`返回`false`，则这个`level`以及更低的其他`level`都打印不了日志。

那么我们怎么修改日志的打印级别呢？可以使用以下命令

```shell
setprop log.tag.<YOUR_LOG_TAG> <LEVEL>

//比如我们的场景则是：
setprop log.tag.MainActivity VERBOSE

```

在huawei上，以上命令可以修改`Log.isLoggable(TAG, Log.VERBOSE)`的返回值，但是打印日志没有起作用，`Log.v(TAG, "VERBOSE")`依然打不出来。而且执行以上命令后，`TimingLogger`也依然打印不出来，因为它打印用的是DEBUG级别。

这是为什么呢？`Log.isLoggable(TAG, Log.VERBOSE)`返回true，理应可以答应VERBOSE级别的日志的。最终发现，针对huawei手机需要这样：拨号`*#*#2846579#*#*`，进入后台设置--LOG设置--勾选AP日志，这样所有日志都可以打印了，注意，这个设置路径在不同huawei手机可能会有区别。

以上介绍的两种方式，无论是`setprop`还是拨号设置的方式，开机重启后都不起效果了。

以上就是我在huawei手机上遇到的问题，没有好的解决方法，首先第一步，打开拨号设置，使得所有日志可以输出，第二步，使用`setprop`修改`isLoggable`的级别，这点很麻烦，只能挨个TAG修改，目前尝试正则匹配所有TAG貌似不成功。

至于`TimingLogger`的使用，其原理比较简单，我们可以基于它的基础上重写一个，然后去除`Log.isLoggable(TAG, Log.VERBOSE)`d的限制，并且修改日志输出级别。





