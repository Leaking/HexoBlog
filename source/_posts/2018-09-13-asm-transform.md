---
title: 从ASM到Android编译插件
date: 2018-09-13 23:12:06
toc: true
tags: Android, JVM
---

AOP(Aspect-oriented programming)技术在Android应用开发中的应用并不少见，如今也并不神秘，最常见的一个场景是插入字节码统计每个方法的运行时间，从而更精准地定位卡顿代码，最常见的实现方式就是基于Javasist + Gradle Transform编写一个Android编译插件处理所有class和jar包，另外一个场景是JakeWharton大神的[hugo组件](https://github.com/JakeWharton/hugo)，这个组件是基于AspectJ开发的，能通过为方法添加Annotation从而统计方法的执行时间和打印方法参数。而这一系列AOP组件基本都存在一些问题，比如，处理字节码速度过慢，编译插件全量处理class，从而导致宿主工程编译时间大大加长。或者有部分class并不能成功注入字节码，或者遇到修改完字节码，后续mergeDex步骤失败，等等问题，不一而足。

以上这些问题我近来都踩了个遍并一一解决，并且实现一个通用的编译插件，可以帮助开发者快速编写插件，修改字节码，实现自己想要的AOP功能，并提供了几个基于这个框架实现的几个实用的AOP组件。这篇文章将围绕我的实现思路展开。

开门见山，我的实现思路是基于Gradle Transform和ASM这两种技术，对应地，这篇文章也只分两部分，第一部分介绍Gradle Transform，接受它在Android编译流程中的应用场景，以及它对class，jar，resource等文件的处理机制，如何基于它实现一个快速的编译插件；第二部分介绍ASM，涉及一些基础的JVM知识，以及如何快速编写ASM代码。另外将穿插我遇到过的各种问题以及问题的答案。


# Transform

要介绍Transform，首先要介绍Android gradle Plugin




