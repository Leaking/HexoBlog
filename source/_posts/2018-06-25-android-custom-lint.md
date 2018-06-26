---
title: Android自定义Lint实践
date: 2018-06-25 20:38:40
tags: Android
---




## 一、自定义Lint在项目中的作用

Lint是Google为Android提供的一套静态代码扫描工具，让你在编码过程中，IDE就能发现并提示代码的质量、安全、性能问题，而无需编译项目或者编写测试用例。Google为Android提供了一系列[自带的Lint规则](https://android.googlesource.com/platform/tools/base/+/master/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks)，这些自带Lint大家日常开发过程中都会遇到，而且，这些Lint也是后面学习写自定义Lint最好的学习素材。

而有时候，我们想要更特殊的代码扫描规则，用来辅助我们的项目框架，那么此时Android自带的Lint规则就无法满足我们的需求，我们需要自己编写自定义Lint。

比如，每个项目都有BaseActivity，很多底层逻辑都和BaseActvity关联，所以基本所有业务页面都需要继承自BaseActvity。而此处为了建立强有效的规范，我们可以通过以下几个方式实现这个目的：

运行时，我们可以借助[Application.ActivityLifecycleCallbacks](https://developer.android.com/reference/android/app/Application.ActivityLifecycleCallbacks)在运行时检测Activiy的类型，如果遇到没有继承自BaseActivity的漏网之鱼，则在Debug模式下抛出异常，提示开发者。

编译期间，我们可以通过检测生成的所有Activity的Class类型，从而在编译期间抛出异常。但是这种方式的工作量相比其他方式较大，投入产出比太低，而且影响编译速度。

编码期间，我们如果使用自定义Lint，则可以在编码期间就发现问题，并给出提示，甚至给出快捷fix的功能。后面我们也会针对这个BaseActivity的场景编写一个自定义Lint。


自定义Lint可以规范页面模块、日志模块等基础组件的使用，有益于团队编码规范，提升代码质量。另外，Lint也经常应用于一些组件库，规范开发者对组件的使用，比如(butterknife-lint)[https://github.com/JakeWharton/butterknife/tree/master/butterknife-lint]。
 


## 二、Lint的实现原理



## 三、如何实现一个自定义Lint



## 四、如何发布一个自定义Lint








