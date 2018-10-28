---
title: 如何使用ASM开发一个快速的Android编译插件
date: 2018-09-13 23:12:06
toc: true
tags: Android, JVM
---

JVM平台上，修改、生成字节码无处不在，从ORM框架（如Hibernate, MyBatis）到Mock框架（如Mockio），再到Java Web中长盛不衰的Spring框架，再到新兴的JVM语言[Kotlin的编译器](https://github.com/JetBrains/kotlin/tree/v1.2.30/compiler/backend/src/org/jetbrains/kotlin/codegen)，都离不开操作字节码，我们能否做XXX事情，这篇文章将分享，，，，





众所周知，JVM平台的语言，它们共同基础都是字节码，而历来JVM平台上的技术，都不乏对字节码操作的场景，

+ 从传统的ORM框架如Hibernate，MyBatis，到比较流行的mock框架，如Mockio，再到JavaEE中长盛不衰的Spring/SprintBoot 框架，背后都有操作字节码的身影。顺便提一句，以上这几个框架基本都使用了cglib这个框架来负责对字节码的处理，而cglib是基于asm实现的。

+ 而除了以上几个框架，很多JVM语言的语法糖，很多都是借助ASM去动态/静态生成字节码的方式实现

+ 再到我们的Android开发中，也不乏操作字节码的身影，而这些大多发生在Android的编译期间，比如lambda语法的[desugar](https://developer.android.com/studio/write/java8-support)过程

操作字节码可以帮我们突破JVM语言本身的很多限制，在代码生成，AOP，性能监控等领域都能起到四两拨千斤的作用。而在Android开发的领域中，我们来探索一下，能否借助字节码技术做一些工具呢？

首先明确我们的目标，我们的目标是*在编译期间，处理所有Android工程中的class文件，jar包文件，甚至资源文件，对其中某些文件进行干预，然后做一些瞒天过海，狸猫换太子的勾当*。


而要达成这个目的，我们需要在原有的编译过程，插入一个我们的任务，在我们的任务中，我们可以接收并处理所有class，jar，而且这一个任务一定要在打包成dex之前，这时候我们就需要引入Transform的概念，我们这篇文章第一部分将介绍Transform的原理，以及在Android原有的编译流程中，被应用到了哪些地方，以及如何优化Transofrm的吞吐量，加快编译速度。而要达成我们的目的，我们除了找到一个时间处理字节码还不够，我们还需要知道怎么处理字节码，常用的框架有ASM, Javasisit，AspectJ，我们将在文章的第二部分介绍对比这几个框架，以及详细介绍ASM的使用，以及部分字节码基础支持，还有我在将ASM应用于Android编译过程中遇到的问题以及解决方案。




















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

