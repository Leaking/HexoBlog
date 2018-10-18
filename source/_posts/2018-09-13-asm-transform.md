---
title: ASM在Android开发中的应用
date: 2018-09-13 23:12:06
toc: true
tags: Android, JVM
---

1、ASM是啥
2、修改class的时机是啥，介绍transform，以及目前Android编译流程中的几个transform
3、重点，大篇幅，，介绍transform的并发，增量
4，解决asm获取android.jar等等坑，凸显hunter为开发者解决了哪些问题
4、速度对比
5、介绍这个hunter transform
6，介绍字节码基础知识（你波浪式，几个操作符）
7，介绍asm基础
8，介绍行号问题，ASM反编译插件
9，介绍四个插件
10，AOP监控
11，增强依赖库的两种方式，1修改实现内容，2修改调用入口（狸猫换太子）
12，DEBUG工具，获取函数入参名的办法




AOP(Aspect-oriented programming)技术在Android应用开发中的应用并不少见，如今也并不神秘，最常见的一个场景是修改/插入字节码，统计每个方法的运行时间，从而更精准地定位卡顿代码，最常见的实现方式就是基于Javasist + Gradle Transform编写一个Android编译插件处理所有class和jar包，另外一个场景是JakeWharton大神的[hugo组件](https://github.com/JakeWharton/hugo)，这个组件是基于AspectJ开发的，能通过为方法添加Annotation从而统计方法的执行时间和打印方法参数。

而这一系列AOP组件基本都存在一些问题，比如，处理字节码速度过慢，编译插件全量处理class，从而导致宿主工程编译时间大大加长。或者有部分class并不能成功注入字节码，或者遇到修改完字节码，后续mergeDex步骤失败，等等问题，不一而足。


以上这些问题我近来都踩了个遍并一一解决，并且实现一个通用的编译插件，可以帮助开发者快速编写插件，修改字节码，实现自己想要的AOP功能，并提供了几个基于这个框架实现的几个实用的AOP组件。这篇文章将围绕我的实现思路展开。

开门见山，我的实现思路是基于Gradle Transform和ASM这两种技术，对应地，这篇文章也只分两部分，第一部分介绍Gradle Transform，接受它在Android编译流程中的应用场景，以及它对class，jar，resource等文件的处理机制，如何基于它实现一个快速的编译插件；第二部分介绍ASM，涉及一些基础的JVM知识，以及如何快速编写ASM代码。另外将穿插我遇到过的各种问题以及问题的答案。



修改字节码的应用场景
1、mocking 框架
2、


# Transform

要介绍Transform，首先要介绍Android gradle Plugin





修改字节码的技术充满想象力，给了我们解决问题的另一个思路，很多性能监控工具，监控内存，UI，数据库，网络等等，基本都是从framework代码入手，通过反射，代理等等hack的方式实现，而究其原因，就是为了突破代码操作权限（比如反射使得访问、修改一个静态变量成为可能），而如果我们直接通过操纵字节码，那么对代码基本获得了最高的权限。

