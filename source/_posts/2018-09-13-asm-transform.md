---
title: 如何使用ASM对你Android项目中的class为所欲为
date: 2018-09-13 23:12:06
toc: true
tags: Android, JVM
---

作为Android开发的你，

是否也曾想过，要对全局所有class插桩，做一些性能监控;
是否也曾遇到，某个第三方依赖，用得不爽，但是不想拿它的源码修改再重新编译，而想对它的class直接做点手脚，比如给Okhttp加一个全局的Interceptor，监控流量？
是否也曾想过，每次写打log时，想让TAG自动生成，让它默认就是当前类的名称，甚至你想让log里自动加上当前代码所在的行数，当同个class中有多行相同日志时，才容易定位日志;
是否也曾想过，Java自带的动态代理太弱了，只能对接口类做动态代理，而我们想对任何类做动态代理;

为了实现上面这些猜想，可能很多人和我一样，第一反应就是能否通过代码生成技术来实现，但是想来想去，APT什么的，貌似不能满足上面的需求，以上这些问题都不能从Java文件入手，而应该从class文件人手。而从class文件入手，我们不得不来近距离接触一下字节码了！

JVM平台上，修改、生成字节码无处不在，从ORM框架（如Hibernate, MyBatis）到Mock框架（如Mockio），再到Java Web中长盛不衰的Spring框架，再到新兴的JVM语言[Kotlin的编译器](https://github.com/JetBrains/kotlin/tree/v1.2.30/compiler/backend/src/org/jetbrains/kotlin/codegen)，还有大名鼎鼎的[cglib](https://github.com/cglib/cglib)项目，都有字节码的声影。

字节码相关技术的强大之处不用多说，而且Android开发中，无论是使用Java开发和Kotlin开发，都是JVM平台的语言，所以如果我们使用字节码技术做一下hack，还可以天然地兼容Java和Kotlin语言，真香。

这篇文章我将介绍如何使用开发一些工具。全文将围绕以下几点展开

+ 如何编写一个快速的编译插件去处理所有class/jar文件
+ 字节码基础知识
+ ASM的使用与常见的问题
+ 几个具体应用案例

话不多说，让我们开始吧


## 如何处理全局的class/jar文件

想要实现这一步，我们需要引出一个概念，就是Android gradle plugin 1.5开始引入的[Transform](http://google.github.io/android-gradle-dsl/javadoc/3.2/)。解析来让我们一起来好好研究下Transform。

我们先从如何使用Transform说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中，让我们看看Transform在[jcenter](https://dl.bintray.com/android/android-tools/com/android/tools/build/transform-api/)上的历个版本。

![](/images/transform_2.png)

所以，很久很久以前我引入transform依赖是这样

```groovy
compile 'com.android.tools.build:transform-api:1.5.0'

```

现在是这样

```groovy
implementation 'com.android.tools.build:gradle-api:3.1.4'  //从2.0.0版本开始就是在gradle-api中了

```




接下来让我们在自定义插件中注册一个自定义Transform


```java
public class CustomPlugin implements Plugin<Project> {

    @SuppressWarnings("NullableProblems")
    @Override
    public void apply(Project project) {
        AppExtension appExtension = (AppExtension)project.getProperties().get("android");
        appExtension.registerTransform(new CustomTransform(), Collections.EMPTY_LIST);
    }

}

```

新建一个自定义Transform


```java
/**
 * 1、Transform的工作:主要是负责将输入的class（可能来自于class文件和jar文件）运输给下一个Transform，运输过程中
 * 你可以对这些class动动手脚，改改字节码。而Transform的输出路径，通过outputProvider获取
 * 2、Transform输入的来源可以通过Scope的概念指定，
 * 3、Transform输入的具体文件类型可以通过ContentType指定
 * 4、Transform可以指定是否支持增量编译，如果增量编译，每次编译，Android编译系统会告诉当前Transform目前哪些文件发生了变化，以及发生
 * 什么变化。
 * 5、Transform的每个输入，并不一定要输出到下一个Transform，它也可以只获取输入，而不输出。输入其实分为两种，一种是消费型输入，需要输出到下个Transform，
 * 一种是引用型输入，getScope方法返回的就是消费型输入，getReferencedScopes方法返回的就是引用型输入。
 */
public class CustomTransform extends Transform {

    public static final String TAG = "CustomTransform";

    public CustomTransform() {
        super();
    }

    @Override
    public String getName() {
        return "CustomTransform";
    }

    @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);
        //消费型输入，可以从中获取jar包和class文件夹路径。需要输出给下一个任务
        Collection<TransformInput> inputs = transformInvocation.getInputs();
        //引用型输入，无需输出。
        Collection<TransformInput> referencedInputs = transformInvocation.getReferencedInputs();
        //OutputProvider管理输出路径，如果消费型输入为空，你会发现OutputProvider == null
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
        for(TransformInput input : inputs) {
            for(JarInput jarInput : input.getJarInputs()) {
                File dest = outputProvider.getContentLocation(
                        jarInput.getFile().getAbsolutePath(),
                        jarInput.getContentTypes(),
                        jarInput.getScopes(),
                        Format.JAR);
                FileUtils.copyFile(jarInput.getFile(), dest);
            }
            for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
                File dest = outputProvider.getContentLocation(directoryInput.getName(),
                        directoryInput.getContentTypes(), directoryInput.getScopes(),
                        Format.DIRECTORY);
                FileUtils.copyDirectory(directoryInput.getFile(), dest);
            }
        }

    }


    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override
    public Set<QualifiedContent.ContentType> getOutputTypes() {
        return super.getOutputTypes();
    }

    @Override
    public Set<? super QualifiedContent.Scope> getReferencedScopes() {
        return TransformManager.EMPTY_SCOPES;
    }


    @Override
    public Map<String, Object> getParameterInputs() {
        return super.getParameterInputs();
    }

    @Override
    public boolean isCacheable() {
        return false;
    }

    @Override
    public boolean isIncremental() {
        return true;
    }

}


```

我们来按几个关键方法逐一分析Transform的工作原理

1. Transform最重要的工作是负责将输入的class（可能来自于class文件和jar文件）输出给下一个Transform（这一步可以参考上面代码中的transform方法），中间过程你可以对这些class动动手脚，改改字节码。而Transform的输出路径，通过OutputProvider获取。另外，Transform可以获取输入的class，而不输出，这取决于输入类型，输入分为两种，一种是消费型输入，需要输出到下个Transform，另一种是引用型输入，不需要输出到下一个Transform。getScope
方法返回的就是消费型输入，getReferencedScopes方法返回的是引用型输入。当getScope返回为空时，你会发现OutputProvider会是一个空对象。那让我们想一个问题，什么时候我只需要引用型输出呢？其实很常见，比如，Android Studio的Instant Run在判断当前编译的和上次编译的差异时，就需要检查class和jar包发生什么变化，而不需要输出任何东西，此时就只需要引用型输入即可。详见[InstantRunVerifierTransform](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/InstantRunVerifierTransform.java)

另外，我们从transform的代码可以发现，Transform的输入，分为了jar包和文件夹两种类型。

2. Transform输入的来源可以通过Scope的概念指定，

```java
 	enum Scope implements ScopeType {
        /** 主module */
        PROJECT(0x01),
        /** 所有子module */
        SUB_PROJECTS(0x04),
        /** 第三方依赖 */
        EXTERNAL_LIBRARIES(0x10),
        /** 测试代码 */
        TESTED_CODE(0x20),
        /** provided依赖 */
        PROVIDED_ONLY(0x40)
    }
```

一般我们都是组合使用上面这几个类型，[TransformManager](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java)有几个常用的Scope集合方便开发者使用。	


3. Transform输入的具体文件类型可以通过ContentType指定

一般可以使用的类型只有class和jar两种文件类型，注意，DefaultContentType枚举类型中的CLASSES就包含了class文件和jar文件。此处还有另一个枚举类型RESOURCE，我还没确定它的使用方式，貌似并不是代表图片，manifest这些资源文件，而是传统java工程里的resources文件夹下的资源。

```java
enum DefaultContentType implements ContentType {
        /**
         * The content is compiled Java code. This can be in a Jar file or in a folder. If
         * in a folder, it is expected to in sub-folders matching package names.
         */
        CLASSES(0x01),

        /** The content is standard Java resources. */
        RESOURCES(0x02);

        private final int value;

        DefaultContentType(int value) {
            this.value = value;
        }

        @Override
        public int getValue() {
            return value;
        }
    }
```
在另一个枚举类中，还有一些隐藏类型，比如DEX文件，不过这些只有Android编译器可以使用。可以详见[com.android.build.gradle.internal.pipeline.ExtendedContentType](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/internal/pipeline/ExtendedContentType.java?autodive=0%2F%2F%2F)，[TransformManager](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java)有几个常用的ContentType集合方便开发者使用，而我们字节码处理一般使用`TransformManager.CONTENT_CLASS`



4. Transform可以指定是否支持增量编译，如果增量编译，每次编译，Android编译系统会告诉当前Transform目前哪些文件发生了变化，以及发生
什么变化。
5. Transform的每个输入，并不一定要输出到下一个Transform，它也可以只获取输入，而不输出。输入其实分为两种，一种是消费型输入，需要输出到下个Transform，
一种是引用型输入，getScope方法返回的就是消费型输入，getReferencedScopes方法返回的就是引用型输入。


该方法在 javaCompile 之后调用， 会遍历所有的 transform，然后一一添加进 TransformManager。 加完自定义的 Transform 之后，再添加 Proguard, JarMergeTransform, MultiDex, Dex 等 Transform。


1、Transform的原理


Input
SecondaryInput
ReferencedInput
ParameterInput


1、Android的编译过程，把Transform应用到什么地方
2、如何使用自定义Transform





首先得让我们来看看Android编译过程，proguard，java 8语法的desugar等等步骤其实都是需要全局处理所有class/jar文件，那么它们是怎么实现的呢？这个时候就需要引入一个概念：[Transform](http://google.github.io/android-gradle-dsl/javadoc/3.2/)，





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

