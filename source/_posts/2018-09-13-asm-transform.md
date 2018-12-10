---
title: 一起玩转Android项目中的字节码
date: 2018-09-13 23:12:06
tags: Android, JVM
---


![](/images/asm_logo.jpeg)


带你对项目中的字节码为所欲为

<!-- more -->


作为Android开发，日常写Java代码之余，是否想过，玩玩class文件？直接对class文件的字节码下手，我们可以做很多好玩的事情，比如：

+ 对全局所有class插桩，做UI，内存，网络等等方面的性能监控
+ 发现某个第三方依赖，用起来不爽，但是不想拿它的源码修改再重新编译，而想对它的class直接做点手脚
+ 每次写打log时，想让TAG自动生成，让它默认就是当前类的名称，甚至你想让log里自动加上当前代码所在的行数，更方便定位日志位置
+ Java自带的动态代理太弱了，只能对接口类做动态代理，而我们想对任何类做动态代理

为了实现上面这些想法，可能我们最开始的第一反应，都是能否通过代码生成技术、APT，抑或反射、抑或动态代理来实现，但是想来想去，貌似这些方案都不能很好满足上面的需求，而且，有些问题不能从Java文件入手，而应该从class文件寻找突破。而从class文件入手，我们就不得不来近距离接触一下字节码！

JVM平台上，修改、生成字节码无处不在，从ORM框架（如Hibernate, MyBatis）到Mock框架（如Mockio），再到Java Web中的常青树Spring框架，再到新兴的JVM语言[Kotlin的编译器](https://github.com/JetBrains/kotlin/tree/v1.2.30/compiler/backend/src/org/jetbrains/kotlin/codegen)，还有大名鼎鼎的[cglib](https://github.com/cglib/cglib)项目，都有字节码的身影。

字节码相关技术的强大之处自然不用多说，而且在Android开发中，无论是使用Java开发和Kotlin开发，都是JVM平台的语言，所以如果我们在Android开发中，使用字节码技术做一下hack，还可以天然地兼容Java和Kotlin语言。

近来我对字节码技术在Android上的应用做了一些调研和实践，顺便做了几个小轮子，项目地址：[Hunter](https://github.com/Leaking/Hunter)

+ Hunter: 一个插件框架，在它的基础上可以快速开发一个并发、增量的字节码编译插件，帮助开发人员隐藏了Transform和ASM的绝大部分逻辑，开发者只需写少量的ASM code，就可以开发一款编译插件，修改Android项目的字节码。

在上面框架基础上，我还开发了几个小工具

 + [OkHttp-Plugin](#okhttp-plugin): 可以为你的应用所有的OkhttpClient设置全局 [Interceptor](https://github.com/square/okhttp/wiki/Interceptors) / [Eventlistener](https://github.com/square/okhttp/wiki/Events)，
(包括第三方依赖里的OkhttpClient)，借助这个插件，可以轻松实现全局网络监控。
 + [Timing-Plugin](#timing-plugin): 帮你监控所有UI线程的执行耗时，并且提供了算法，帮你打印出一个带有每步耗时的堆栈，统计卡顿方法分布，你也可以自定义分析卡顿堆栈的方式。
 + [LogLine-Plugin](#logline-plugin): 为你的日志加上行号
 + [Debug-Plugin](#debug-plugin): 只要为指定方法加上某个annotation，就可以帮你打印出这个方法所有输入参数的值，以及返回值和执行时间(其实，JakeWharton的[hugo](https://github.com/JakeWharton/hugo)用AspectJ实现了类似功能，而我的实现方式是基于ASM，ASM处理字节码的速度更快)
 + 你可以在这里查看我想继续开发的一些插件 [TODO](https://github.com/Leaking/Hunter/blob/master/TODO.md)，另外，欢迎你提供你宝贵的idea


今天写这篇文章，分享自己摸索相关技术和开发这个项目过程中的一些积累。

这个项目主要使用的技术是Android gradle插件，Transform，ASM与字节码基础。这篇文章将主要围绕以下几个技术点展开：

+ Transform的应用、原理、优化
+ ASM的应用，开发流，以及与Android工程的适配
+ 几个具体应用案例

所以阅读这篇文章，读者最好有Android开发以及编写简单Gradle插件的背景知识。

话不多说，让我们开始吧。


# 一、Transform


## 引入Transform


[Transform](http://google.github.io/android-gradle-dsl/javadoc/3.2/)是Android gradle plugin 1.5开始引入的概念。

我们先从如何引入Transform依赖说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中，让我们看看Transform在[jcenter](https://dl.bintray.com/android/android-tools/com/android/tools/build/transform-api/)上的历个版本。

![](/images/transform_2.png)

所以，很久很久以前我引入transform依赖是这样

```groovy
compile 'com.android.tools.build:transform-api:1.5.0'

```

现在是这样

```groovy
//从2.0.0版本开始就是在gradle-api中了
implementation 'com.android.tools.build:gradle-api:3.1.4'  

```


然后，让我们在自定义插件中注册一个自定义Transform，gradle插件可以使用java，groovy，kotlin编写，我这里选择使用java。


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
那么如何写一个自定义Transform呢？


## Transform的原理与应用

介绍如何应用Transform之前，我们先介绍Transform的原理，一图胜千言


![](/images/transformconsume_transform.png)



每个Transform其实都是一个gradle task，Android编译器中的TaskManager将每个Transform串连起来，第一个Transform接收来自javac编译的结果，以及已经拉取到在本地的第三方依赖（jar. aar），还有resource资源，注意，这里的resource并非android项目中的res资源，而是asset目录下的资源。这些编译的中间产物，在Transform组成的链条上流动，每个Transform节点可以对class进行处理再传递给下一个Transform。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个Transform中，而我们自定义的Transform，会插入到这个Transform链条的最前面。


但其实，上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。至于怎么在一个Transform中声明两种输入，以及怎么处理两种输入，后面将有示例代码。


为了印证Transform的工作原理和应用方式，我们也可以从Android gradle plugin源码入手找出证据，在TaskManager中，有一个方法`createPostCompilationTasks`.为了避免贴篇幅太长的源码，这里附上链接

[TaskManager#createPostCompilationTasks](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/internal/TaskManager.java#2154)


这个方法的脉络很清晰，我们可以看到，Jacoco，Desugar，MergeJavaRes，AdvancedProfiling，Shrinker，Proguard, JarMergeTransform, MultiDex, Dex都是通过Transform的形式一个个串联起来。其中也有将我们自定义的Transform插进去。


讲完了Transform的数据流动的原理，我们再来介绍一下Transform的输入数据的过滤机制，Transform的数据输入，可以通过Scope和ContentType两个维度进行过滤。

![](/images/transformscope&contenttype.png)


ContentType，顾名思义，就是数据类型，在插件开发中，我们一般只能使用CLASSES和RESOURCES两种类型，注意，其中的CLASSES已经包含了class文件和jar文件


![](/images/transformContentType.png)

从图中可以看到，除了CLASSES和RESOURCES，还有一些我们开发过程无法使用的类型，比如DEX文件，这些隐藏类型在一个独立的枚举类[ExtendedContentType](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/internal/pipeline/ExtendedContentType.java?autodive=0%2F%2F%2F)中，这些类型只能给Android编译器使用。另外，我们一般使用[TransformManager](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java)中提供的几个常用的ContentType集合和Scope集合，如果是要处理所有class和jar的字节码，ContentType我们一般使用`TransformManager.CONTENT_CLASS`。


Scope相比ContentType则是另一个维度的过滤规则，

![](/images/transformScope.png)


我们可以发现，左边几个类型可供我们使用，而我们一般都是组合使用这几个类型，[TransformManager](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/gradle-core/src/main/groovy/com/android/build/gradle/internal/transforms/ShrinkResourcesTransform.java)有几个常用的Scope集合方便开发者使用。
如果是要处理所有class字节码，Scope我们一般使用`TransformManager.SCOPE_FULL_PROJECT`。


好，目前为止，我们介绍了Transform的数据流动的原理，输入的类型和过滤机制，我们再写一个简单的自定义Transform，让我们对Transform可以有一个更具体的认识


```java
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
        //当前是否是增量编译
        boolean isIncremental = transformInvocation.isIncremental();
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
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了        
                FileUtils.copyFile(jarInput.getFile(), dest);
            }
            for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
                File dest = outputProvider.getContentLocation(directoryInput.getName(),
                        directoryInput.getContentTypes(), directoryInput.getScopes(),
                        Format.DIRECTORY);
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了        
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
        return true;
    }

	
    @Override 
    public boolean isIncremental() {
        return true; //是否开启增量编译
    }

}


```

可以看到，在transform方法中，我们将每个jar包和class文件复制到dest路径，这个dest路径就是下一个Transform的输入数据，而在复制时，我们就可以做一些狸猫换太子，偷天换日的事情了，先将jar包和class文件的字节码做一些修改，再进行复制即可，至于怎么修改字节码，就要借助我们后面介绍的ASM了。而如果开发过程要看你当前transform处理之后的class/jar包，可以到
/build/intermediates/transforms/CustomTransform/下查看，你会发现所有jar包命名都是123456递增，这是正常的，这里的命名规则可以在OutputProvider.getContentLocation的具体实现中找到

```java

public synchronized File getContentLocation(
        @NonNull String name,
        @NonNull Set<ContentType> types,
        @NonNull Set<? super Scope> scopes,
        @NonNull Format format) {
    // runtime check these since it's (indirectly) called by 3rd party transforms.
    checkNotNull(name);
    checkNotNull(types);
    checkNotNull(scopes);
    checkNotNull(format);
    checkState(!name.isEmpty());
    checkState(!types.isEmpty());
    checkState(!scopes.isEmpty());

    // search for an existing matching substream.
    for (SubStream subStream : subStreams) {
        // look for an existing match. This means same name, types, scopes, and format.
        if (name.equals(subStream.getName())
                && types.equals(subStream.getTypes())
                && scopes.equals(subStream.getScopes())
                && format == subStream.getFormat()) {
            return new File(rootFolder, subStream.getFilename());
        }
    }
    //按位置递增！！	
    // didn't find a matching output. create the new output
    SubStream newSubStream = new SubStream(name, nextIndex++, scopes, types, format, true);

    subStreams.add(newSubStream);

    return new File(rootFolder, newSubStream.getFilename());
}

```


## Transform的优化：增量与并发

到此为止，看起来Transform用起来也不难，但是，如果直接这样使用，会大大拖慢编译时间，为了解决这个问题，摸索了一段时间后，也借鉴了Android编译器中Desugar等几个Transform的实现，发现我们可以使用增量编译，并且上面transform方法遍历处理每个jar/class的流程，其实可以并发处理，加上一般编译流程都是在PC上，所以我们可以尽量敲诈机器的资源。

想要开启增量编译，我们需要重写Transform的这个接口，返回true。


```java
@Override 
public boolean isIncremental() {
    return true;
}

```


虽然开启了增量编译，但也并非每次编译过程都是支持增量的，毕竟一次clean build完全没有增量的基础，所以，我们需要检查当前编译是否是增量编译。

如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理
如果是增量编译，则要检查每个文件的Status，Status分四种，并且对这四种文件的操作也不尽相同

![](/images/transform_Status.png)

+ NOTCHANGED: 当前文件不需处理，甚至复制操作都不用；
+ ADDED、CHANGED: 正常处理，输出给下一个任务；
+ REMOVED: 移除outputProvider获取路径对应的文件。

大概实现可以一起看看下面的代码


```java 

@Override
public void transform(TransformInvocation transformInvocation){
    Collection<TransformInput> inputs = transformInvocation.getInputs();
    TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
    boolean isIncremental = transformInvocation.isIncremental();
    //如果非增量，则清空旧的输出内容
    if(!isIncremental) {
        outputProvider.deleteAll();
    }	
    for(TransformInput input : inputs) {
        for(JarInput jarInput : input.getJarInputs()) {
            Status status = jarInput.getStatus();
            File dest = outputProvider.getContentLocation(
                    jarInput.getName(),
                    jarInput.getContentTypes(),
                    jarInput.getScopes(),
                    Format.JAR);
            if(isIncremental && !emptyRun) {
                switch(status) {
                    case NOTCHANGED:
                        continue;
                    case ADDED:
                    case CHANGED:
                        transformJar(jarInput.getFile(), dest, status);
                        break;
                    case REMOVED:
                        if (dest.exists()) {
                            FileUtils.forceDelete(dest);
                        }
                        break;
                }
            } else {
                transformJar(jarInput.getFile(), dest, status);
            }
        }

        for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
            File dest = outputProvider.getContentLocation(directoryInput.getName(),
                    directoryInput.getContentTypes(), directoryInput.getScopes(),
                    Format.DIRECTORY);
            FileUtils.forceMkdir(dest);
            if(isIncremental && !emptyRun) {
                String srcDirPath = directoryInput.getFile().getAbsolutePath();
                String destDirPath = dest.getAbsolutePath();
                Map<File, Status> fileStatusMap = directoryInput.getChangedFiles();
                for (Map.Entry<File, Status> changedFile : fileStatusMap.entrySet()) {
                    Status status = changedFile.getValue();
                    File inputFile = changedFile.getKey();
                    String destFilePath = inputFile.getAbsolutePath().replace(srcDirPath, destDirPath);
                    File destFile = new File(destFilePath);
                    switch (status) {
                        case NOTCHANGED:
                            break;
                        case REMOVED:
                            if(destFile.exists()) {
                                FileUtils.forceDelete(destFile);
                            }
                            break;
                        case ADDED:
                        case CHANGED:
                            FileUtils.touch(destFile);
                            transformSingleFile(inputFile, destFile, srcDirPath);
                            break;
                    }
                }
            } else {
                transformDir(directoryInput.getFile(), dest);
            }

        }
    }
}

```
这就能为我们的编译插件提供增量的特性。

实现了增量编译后，我们最好也支持并发编译，并发编译的实现并不复杂，只需要将上面处理单个jar/class的逻辑，并发处理，最后阻塞等待所有任务结束即可。


```java

private WaitableExecutor waitableExecutor = WaitableExecutor.useGlobalSharedThreadPool();


//异步并发处理jar/class
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveJar(srcJar, destJar);
    return null;
});
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveSingleClassToFile(file, outputFile, inputDirPath);
    return null;
});  


//等待所有任务结束
waitableExecutor.waitForTasksWithQuickFail(true);

```


接下来我们对编译速度做一个对比，每个实验都是5次同种条件下编译10次，去除最大大小值，取平均时间

首先，在我工作中的项目（Java/kotlin代码量行数百万级别），我们先做一次cleanbuild

```groovy
./gradlew clean assembleDebug --profile
```

给项目中添加UI耗时统计，全局每个方法（包括普通class文件和第三方jar包中的所有class）的第一行和最后一行都进行插桩，实现方式就是Transform+ASM，对比一下并发Transform和非并发Transform下，Tranform这一步的耗时

![](/images/transform_time_1.png)

可以发现，并发编译，基本比非并发编译速度提高了80%。效果很显著。


然后，让我们再做另一个试验，我们在项目中模拟日常修改某个class文件的一行代码，这时是符合增量编译的环境的。然后在刚才基础上还是做同样的插桩逻辑，对比增量Transform和全量Transform的差异。

```groovy
./gradlew assembleDebug --profile
```

![](/images/transform_time_2.png)



可以发现，增量的速度比全量的速度提升了3倍多，而且这个速度优化会随着工程的变大而更加显著。

数据表明，增量和并发对编译速度的影响是很大的。而我在查看Android gradle plugin自身的十几个Transform时，发现它们实现方式也有一些区别，有些用kotlin写，有些用java写，有些支持增量，有些不支持，而且是代码注释写了一个大大的FIXME, To support incremental build。所以，讲道理，现阶段的Android编译速度，还是有提升空间的。


上面我们介绍了Transform，以及如何高效地在编译期间处理所有字节码，那么具体怎么处理字节码呢？接下来让我们一起看看JVM平台上的处理字节码神兵利器，ASM!


# 二、ASM

ASM的官网在这里[https://asm.ow2.io/](https://asm.ow2.io/)，贴一下它的主页介绍，一起感受下它的强大


![](/images/transform_asm_introduce.png)


JVM平台上，处理字节码的框架最常见的就三个，ASM，Javasist，AspectJ。我尝试过Javasist，而AspectJ也稍有了解，最终选择ASM，因为使用它可以更底层地处理字节码的每条命令，处理速度、内存占用，也优于其他两个框架。


我们可以来做一个对比，上面我们所做的计算编译时间实验的基础上，做如下试验，分别用ASM和Javasist全量处理工程所有class，并且都不开启并发处理的情况下，一次clean build中，transform的耗时对比如下

![](/images/transform_asm_javasist.png)

ASM相比Javasist的优势非常显著，ASM相比其他字节码操作库的效率和性能优势应该毋庸置疑的，毕竟是诸多JVM语言钦定的字节码生成库。

我们这部分将来介绍ASM，但是由于篇幅问题，不会从字节码的基础展开介绍，会通过几个实例的实现介绍一些字节码的相关知识，另外还会介绍ASM的使用，以及ASM解析class文件结构的原理，还有应用于Android插件开发时，遇到的问题，及其解决方案。


## ASM的引入

下面是一份完整的gradle自定义plugin + transform + asm所需依赖，注意一下，此处两个gradleApi的区别

```gradle 

dependencies {

    //使用项目中指定的gradle wrapper版本，插件中使用的Project对象等等就来自这里
    implementation gradleApi()     

    //使用本地的groovy
    implementation localGroovy()   

    //Android编译的大部分gradle源码，比如上面讲到的TaskManager
    implementation 'com.android.tools.build:gradle:3.1.4'    

    //这个依赖里其实主要存了transform的依赖，注意，这个依赖不同于上面的gradleApi()
    implementation 'com.android.tools.build:gradle-api:3.1.4'   

    //ASM相关
    implementation 'org.ow2.asm:asm:5.1'                        
    implementation 'org.ow2.asm:asm-util:5.1'                    
    implementation 'org.ow2.asm:asm-commons:5.1'   

}


```


### ASM的应用


ASM设计了两种API类型，一种是Tree API, 一种是基于Visitor API(visitor pattern)，

Tree API将class的结构读取到内存，构建一个树形结构，然后需要处理Method、Field等元素时，到树形结构中定位到某个元素，进行操作，然后把操作再写入新的class文件。

Visitor API则将通过接口的方式，分离读class和写class的逻辑，一般通过一个ClassReader负责读取class字节码，然后ClassReader通过一个ClassVisitor接口，将字节码的每个细节按顺序通过接口的方式，传递给ClassVisitor（你会发现ClassVisitor中有多个visitXXXX接口），这个过程就像ClassReader带着ClassVisitor游览了class字节码的每一个指令。


上面这两种解析文件结构的方式在很多处理结构化数据时都常见，一般得看需求背景选择合适的方案，而我们的需求是这样的，出于某个目的，寻找class文件中的一个hook点，进行字节码修改，这种背景下，我们选择Visitor API的方式比较合适。

让我们来写一个简单的demo，这段代码很简单，通过Visitor API读取一个class的内容，保存到另一个文件


```java

private void copy(String inputPath, String outputPath) {
    try {
        FileInputStream is = new FileInputStream(inputPath);
        ClassReader cr = new ClassReader(is);
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        cr.accept(cw, 0);
        FileOutputStream fos = new FileOutputStream(outputPath);
        fos.write(cw.toByteArray());
        fos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

```
首先，我们通过ClassReader读取某个class文件，然后定义一个ClassWriter，这个ClassWriter我们可以看它源码，其实就是一个ClassVisitor的实现，负责将ClassReader传递过来的数据写到一个字节流中，而真正触发这个逻辑就是通过ClassWriter的accept方式。


```java

public void accept(ClassVisitor classVisitor, Attribute[] attributePrototypes, int parsingOptions) {
    
    // 读取当前class的字节码信息
    int accessFlags = this.readUnsignedShort(currentOffset);
    String thisClass = this.readClass(currentOffset + 2, charBuffer);
    String superClass = this.readClass(currentOffset + 4, charBuffer);
    String[] interfaces = new String[this.readUnsignedShort(currentOffset + 6)];

    
    
    //classVisitor就是刚才accept方法传进来的ClassWriter，每次visitXXX都负责将字节码的信息存储起来
    classVisitor.visit(this.readInt(this.cpInfoOffsets[1] - 7), accessFlags, thisClass, signature, superClass, interfaces);
    
    /**
        略去很多visit逻辑
    */

    //visit Attribute
    while(attributes != null) {
        Attribute nextAttribute = attributes.nextAttribute;
        attributes.nextAttribute = null;
        classVisitor.visitAttribute(attributes);
        attributes = nextAttribute;
    }

    /**
        略去很多visit逻辑
    */

    classVisitor.visitEnd();
}


```




最后，我们通过ClassWriter的toByteArray()，将从ClassReader传递到ClassWriter的字节码导出，写入新的文件即可。这就完成了class文件的复制，这个demo虽然很简单，但是涵盖了ASM使用Visitor API修改字节码最底层的原理，大致流程如图

![](/images/transform_ASM-1.png)


我们来分析一下，不难发现，如果我们要修改字节码，就是要从ClassWriter入手，上面我们提到ClassWriter中每个visitXXX（这些接口实现自ClassVisitor）都会保存字节码信息并最终可以导出，那么如果我们可以代理ClassWriter的接口，就可以干预最终字节码的生成了。

那么上面的图就应该是这样

![](/images/transform_ASM-2.png)


我们只要稍微看一下ClassVisitor的代码，发现它的构造函数，是可以接收另一个ClassVisitor的，从而通过这个ClassVisitor代理所有的方法。让我们来看一个例子，为class中的每个方法调用语句的开头和结尾插入一行代码

修改前的方法是这样

```java

private static void printTwo() {
    printOne();
    printOne();
}   

```

被修改后的方法是这样


```java

private static void printTwo() {
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
}

```

让我们来看一下如何用ASM实现

```java
private static void weave(String inputPath, String outputPath) {
    try {
        FileInputStream is = new FileInputStream(inputPath);
        ClassReader cr = new ClassReader(is);
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        CallClassAdapter adapter = new CallClassAdapter(cw);
        cr.accept(adapter, 0);
        FileOutputStream fos = new FileOutputStream(outputPath);
        fos.write(cw.toByteArray());
        fos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

```

这段代码和上面的实现复制class的代码唯一区别就是，使用了CallClassAdapter，它是一个自定义的ClassVisitor，我们将ClassWriter传递给CallClassAdapter的构造函数。来看看它的实现

```java

//CallClassAdapter.java
public class CallClassAdapter extends ClassVisitor implements Opcodes {

    public CallClassAdapter(final ClassVisitor cv) {
        super(ASM5, cv);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
    }

    @Override
    public MethodVisitor visitMethod(final int access, final String name,
                                     final String desc, final String signature, final String[] exceptions) {
        MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);
        return mv == null ? null : new CallMethodAdapter(name, mv);
    }

}

//CallMethodAdapter.java
class CallMethodAdapter extends MethodVisitor implements Opcodes {

    public CallMethodAdapter(final MethodVisitor mv) {
        super(ASM5, mv);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {

        mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("CALL " + name);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

        mv.visitMethodInsn(opcode, owner, name, desc, itf);

        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("RETURN " + name);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

    }

}

```

CallClassAdapter中的visitMethod使用了一个自定义的MethodVisitor-----CallMethodAdapter，它也是代理了原来的MethodVisitor，原理和ClassVisitor的代理一样。

看到这里，貌似使用ASM修改字节码的大概套路都走完了，那么如何写出上面`visitMethodInsn`方法中插入打印方法名的逻辑，这就需要一些字节码的基础知识了，我们说过这里不会展开介绍字节码，但是我们可以介绍一些快速学习字节码的方式，同时也是开发字节码相关工程一些实用的工具。

在这之前，我们先讲讲行号的问题

### 如何验证行号

上面我们给每一句方法调用的前后都插入了一行日志打印，那么有没有想过，这样岂不是打乱了代码的行数，这样，万一crash了，定位堆栈岂不是乱套了。其实并不然，在上面visitMethodInsn中做的东西，其实都是在同一行中插入的代码，上面我们贴出来的代码是这样

```java

private static void printTwo() {
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
}

```

无论你用idea还是eclipse打开上面的class文件，都是一行行展示的，但是其实class内部真实的行数应该是这样

```java

private static void printTwo() {
    System.out.println("CALL printOne"); printOne(); System.out.println("RETURN printOne");
    System.out.println("CALL printOne"); printOne(); System.out.println("RETURN printOne");
}

```

idea下可以开启一个选项，让查看class内容时，保留真正的行数

开启后，你看到的是这样

![](/images/transform_line_2.png)

我们可以发现，17行和18行，分别包含了三句代码。

而开启选项之前是这样

![](/images/transform_line_1.png)

那么如何开启这个选项呢？Mac下`cmd + shift + A`输入Registry，勾选这两个选项

![](/images/transform_line_3.png)

其实无论字节码和ASM的代码上看，class中的所有代码，都是先声明行号X，然后开始几条字节码指令，这几条字节码对应的代码都在行号X中，直到声明下一个新的行号。

## ASM code

解析来介绍，如何写出上面生成代码的逻辑。首先，我们设想一下，如果要对某个class进行修改，那需要对字节码具体做什么修改呢？最直观的方法就是，先编译生成目标class，然后看它的字节码和原来class的字节码有什么区别（查看字节码可以使用javap工具），但是这样还不够，其实我们最终并不是读写字节码，而是使用ASM来修改，我们这里先做一个区别，bytecode vs ASM code，前者就是JVM意义的字节码，而后者是用ASM描述的bytecode，其实二者非常的接近，只是ASM code用Java代码来描述。所以，我们应该是对比ASM code，而不是对比bytecode。对比ASM code的diff，基本就是我们要做的修改。

而ASM也提供了一个这样的类：ASMifier，它可以生成ASM code，但是，其实还有更快捷的工具，Intellij IDEA有一个插件
[Asm Bytecode Outline](https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline)，可以查看一个class文件的bytecode和ASM code。


到此为止，貌似使用对比ASM code的方式，来实现字节码修改也不难，但是，这种方式只是可以实现一些修改字节码的基础场景，还有很多场景是需要对字节码有一些基础知识才能做到，而且，要阅读懂ASM code，也是需要一定字节码的的知识。所以，如果要开发字节码工程，还是需要学习一番字节码。


## ClassWriter在Android上的坑

如果我们直接按上面的套路，将ASM应用到Android编译插件中，会踩到一个坑，这个坑来自于ClassWriter，具体是因为ClassWriter其中的一个逻辑，寻找两个类的共同父类。可以看看ClassWriter中的这个方法getCommonSuperClass，



```java

/**
 * Returns the common super type of the two given types. The default
 * implementation of this method <i>loads</i> the two given classes and uses
 * the java.lang.Class methods to find the common super class. It can be
 * overridden to compute this common super type in other ways, in particular
 * without actually loading any class, or to take into account the class
 * that is currently being generated by this ClassWriter, which can of
 * course not be loaded since it is under construction.
 * 
 * @param type1
 *            the internal name of a class.
 * @param type2
 *            the internal name of another class.
 * @return the internal name of the common super class of the two given
 *         classes.
 */
protected String getCommonSuperClass(final String type1, final String type2) {
    Class<?> c, d;
    ClassLoader classLoader = getClass().getClassLoader();
    try {
        c = Class.forName(type1.replace('/', '.'), false, classLoader);
        d = Class.forName(type2.replace('/', '.'), false, classLoader);
    } catch (Exception e) {
        throw new RuntimeException(e.toString());
    }
    if (c.isAssignableFrom(d)) {
        return type1;
    }
    if (d.isAssignableFrom(c)) {
        return type2;
    }
    if (c.isInterface() || d.isInterface()) {
        return "java/lang/Object";
    } else {
        do {
            c = c.getSuperclass();
        } while (!c.isAssignableFrom(d));
        return c.getName().replace('.', '/');
    }
}


```


这个方法用于寻找两个类的共同父类，我们可以看到它是获取当前class的classLoader加载两个输入的类型，而编译期间使用的classloader并没有加载Android项目中的代码，所以我们需要一个自定义的ClassLoader，将前面提到的Transform中接收到的所有jar以及class，还有android.jar都添加到自定义ClassLoader中。（其实上面这个方法注释中已经暗示了这个方法存在的一些问题）

如下

```java

public static URLClassLoader getClassLoader(Collection<TransformInput> inputs,
                                            Collection<TransformInput> referencedInputs,
                                            Project project) throws MalformedURLException {
    ImmutableList.Builder<URL> urls = new ImmutableList.Builder<>();
    String androidJarPath  = getAndroidJarPath(project);
    File file = new File(androidJarPath);
    URL androidJarURL = file.toURI().toURL();
    urls.add(androidJarURL);
    for (TransformInput totalInputs : Iterables.concat(inputs, referencedInputs)) {
        for (DirectoryInput directoryInput : totalInputs.getDirectoryInputs()) {
            if (directoryInput.getFile().isDirectory()) {
                urls.add(directoryInput.getFile().toURI().toURL());
            }
        }
        for (JarInput jarInput : totalInputs.getJarInputs()) {
            if (jarInput.getFile().isFile()) {
                urls.add(jarInput.getFile().toURI().toURL());
            }
        }
    }
    ImmutableList<URL> allUrls = urls.build();
    URL[] classLoaderUrls = allUrls.toArray(new URL[allUrls.size()]);
    return new URLClassLoader(classLoaderUrls);
}

```

但是，如果只是替换了getCommonSuperClass中的Classloader，依然还有一个更深的坑，我们可以看看前面getCommonSuperClass的实现，它是如何寻找父类的呢？它是通过Class.forName加载某个类，然后再去寻找父类，但是，但是，android.jar中的类可不能随随便便加载的呀，android.jar对于Android工程来说只是编译时依赖，运行时是用Android机器上自己的android.jar。而且android.jar所有方法包括构造函数都是空实现，其中都只有一行代码

```java
throw new RuntimeException("Stub!");
```

这样加载某个类时，它的静态域就会被触发，而如果有一个static的变量刚好在声明时被初始化，而初始化中只有一个RuntimeException，此时就会抛异常。

所以，我们不能通过这种方式来获取父类，能否通过不需要加载class就能获取它的父类的方式呢？谜底就在眼前，父类其实也是一个class的字节码中的一项数据，那么我们就从字节码中查询父类即可。最终实现是这样。


```java

public class ExtendClassWriter extends ClassWriter {

    public static final String TAG = "ExtendClassWriter";

    private static final String OBJECT = "java/lang/Object";

    private ClassLoader urlClassLoader;

    public ExtendClassWriter(ClassLoader urlClassLoader, int flags) {
        super(flags);
        this.urlClassLoader = urlClassLoader;
    }

    @Override
    protected String getCommonSuperClass(final String type1, final String type2) {
        if (type1 == null || type1.equals(OBJECT) || type2 == null || type2.equals(OBJECT)) {
            return OBJECT;
        }

        if (type1.equals(type2)) {
            return type1;
        }

        ClassReader type1ClassReader = getClassReader(type1);
        ClassReader type2ClassReader = getClassReader(type2);
        if (type1ClassReader == null || type2ClassReader == null) {
            return OBJECT;
        }

        if (isInterface(type1ClassReader)) {
            String interfaceName = type1;
            if (isImplements(interfaceName, type2ClassReader)) {
                return interfaceName;
            }
            if (isInterface(type2ClassReader)) {
                interfaceName = type2;
                if (isImplements(interfaceName, type1ClassReader)) {
                    return interfaceName;
                }
            }
            return OBJECT;
        }

        if (isInterface(type2ClassReader)) {
            String interfaceName = type2;
            if (isImplements(interfaceName, type1ClassReader)) {
                return interfaceName;
            }
            return OBJECT;
        }

        final Set<String> superClassNames = new HashSet<String>();
        superClassNames.add(type1);
        superClassNames.add(type2);

        String type1SuperClassName = type1ClassReader.getSuperName();
        if (!superClassNames.add(type1SuperClassName)) {
            return type1SuperClassName;
        }

        String type2SuperClassName = type2ClassReader.getSuperName();
        if (!superClassNames.add(type2SuperClassName)) {
            return type2SuperClassName;
        }

        while (type1SuperClassName != null || type2SuperClassName != null) {
            if (type1SuperClassName != null) {
                type1SuperClassName = getSuperClassName(type1SuperClassName);
                if (type1SuperClassName != null) {
                    if (!superClassNames.add(type1SuperClassName)) {
                        return type1SuperClassName;
                    }
                }
            }

            if (type2SuperClassName != null) {
                type2SuperClassName = getSuperClassName(type2SuperClassName);
                if (type2SuperClassName != null) {
                    if (!superClassNames.add(type2SuperClassName)) {
                        return type2SuperClassName;
                    }
                }
            }
        }

        return OBJECT;
    }

    private boolean isImplements(final String interfaceName, final ClassReader classReader) {
        ClassReader classInfo = classReader;

        while (classInfo != null) {
            final String[] interfaceNames = classInfo.getInterfaces();
            for (String name : interfaceNames) {
                if (name != null && name.equals(interfaceName)) {
                    return true;
                }
            }

            for (String name : interfaceNames) {
                if(name != null) {
                    final ClassReader interfaceInfo = getClassReader(name);
                    if (interfaceInfo != null) {
                        if (isImplements(interfaceName, interfaceInfo)) {
                            return true;
                        }
                    }
                }
            }

            final String superClassName = classInfo.getSuperName();
            if (superClassName == null || superClassName.equals(OBJECT)) {
                break;
            }
            classInfo = getClassReader(superClassName);
        }

        return false;
    }

    private boolean isInterface(final ClassReader classReader) {
        return (classReader.getAccess() & Opcodes.ACC_INTERFACE) != 0;
    }


    private String getSuperClassName(final String className) {
        final ClassReader classReader = getClassReader(className);
        if (classReader == null) {
            return null;
        }
        return classReader.getSuperName();
    }

    private ClassReader getClassReader(final String className) {
        InputStream inputStream = urlClassLoader.getResourceAsStream(className + ".class");
        try {
            if (inputStream != null) {
                return new ClassReader(inputStream);
            }
        } catch (IOException ignored) {
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException ignored) {
                }
            }
        }
        return null;
    }
}

```

到此为止，我们介绍了在Android上实现修改字节码的两个基础技术Transform+ASM，介绍了其原理和应用，分析了性能优化以及在Android平台上的适配等。在此基础上，我抽象出一个轮子，让开发者写字节码插件时，只需要写少量的ASM code即可，而不需关心Transform和ASM背后的很多细节。详见

https://github.com/Leaking/Hunter/wiki/Developer-API


万事俱备，只欠写一个插件来玩玩了，让我们来看看几个应用案例。


## 应用案例

先抛结论，修改字节码其实也有套路，一种是hack代码调用，一种是hack代码实现。

比如修改Android Framework（android.jar）的实现，你是没办法在编译期间达到这个目的的，因为最终Android Framework的class在Android设备上。所以这种情况下你需要从hack代码调用入手，比如Log.i(TAG, "hello")，你不可能hack其中的实现，但是你可以把它hack成HackLog.i(TAG, "seeyou")。

而如果是要修改第三方依赖或者工程中写的代码，则可以直接hack代码实现，但是，当如果你要插入的字节码比较多时，也可以通过一定技巧减少写ASM code的量，你可以将大部分可以抽象的逻辑抽象到某个写好的class中，然后ASM code只需写调用这个写好的class的语句。

当然上面只是目前按照我的经验做的一点总结，还是有一些更复杂的情况要具体情况具体分析，比如在实现类似JakeWharton的[hugo](https://github.com/JakeWharton/hugo)的功能时，在代码开头获取方法参数名时我就遇到棘手的问题（用了一种二次扫描的方式解决了这个问题，可以移步项目主页参考具体实现）。


我们这里挑选OkHttp-Plugin的实现进行分析、演示如何使用Huntet框架开发一个字节码编译插件。


使用OkHttp的人知道，OkHttp里每一个OkHttp都可以设置自己独立的Intercepter/Dns/EventListener(EventListener是okhttp3.11新增)，但是需要对全局所有OkHttp设置统一的Intercepter/Dns/EventListener就很麻烦，需要一处处设置，而且一些第三方依赖中的OkHttp很大可能无法设置。曾经在官方repo提过这个问题的[issue](https://github.com/square/okhttp/issues/4228)，没有得到很好的回复，作者之一觉得如果是他，他会用依赖注入的方式来实现统一的Okhttp配置，但是这种方式只能说可行但是不理想，后台在reddit发[帖子](https://www.reddit.com/r/androiddev/comments/9nvg0f/a_plugin_framework_to_moidfy_bytecode_of_andrid/)安利自己Hunter这个轮子时，JakeWharton大佬竟然亲自回答了，虽然面对大佬，不过还是要正面刚！争论一波之后，总结一下他的立场，大概如下

> 他觉得我说的好像这是okhttp的锅，然而这其实是okhttp的一个feature，他觉得全局状态是一种不好的编码，所以在设计okhttp没有提供全局Intercepter/Dns/EventListener的接口。而第三方依赖库不能设置自定义Intercepter/Dns/EventListener这是它们的锅。

但是，他的观点我不完全同意，虽然全局状态确实是一种不好的设计，但是，如果要做性能监控之类的功能，这就很难避免或多或少的全局侵入。（不过我确实措辞不当，说得这好像是Okhttp的锅一样）

言归正传，来看看我们要怎么来对OkHttp动刀，请看以下代码


```java

public Builder(){
    this.dispatcher = new Dispatcher();
    this.protocols = OkHttpClient.DEFAULT_PROTOCOLS;
    this.connectionSpecs = OkHttpClient.DEFAULT_CONNECTION_SPECS;
    this.eventListenerFactory = EventListener.factory(EventListener.NONE);
    this.proxySelector = ProxySelector.getDefault();
    this.cookieJar = CookieJar.NO_COOKIES;
    this.socketFactory = SocketFactory.getDefault();
    this.hostnameVerifier = OkHostnameVerifier.INSTANCE;
    this.certificatePinner = CertificatePinner.DEFAULT;
    this.proxyAuthenticator = Authenticator.NONE;
    this.authenticator = Authenticator.NONE;
    this.connectionPool = new ConnectionPool();
    this.dns = Dns.SYSTEM;
    this.followSslRedirects = true;
    this.followRedirects = true;
    this.retryOnConnectionFailure = true;
    this.connectTimeout = 10000;
    this.readTimeout = 10000;
    this.writeTimeout = 10000;
    this.pingInterval = 0;
    this.eventListenerFactory = OkHttpHooker.globalEventFactory;
    this.dns = OkHttpHooker.globalDns;
    this.interceptors.addAll(OkHttpHooker.globalInterceptors);
    this.networkInterceptors.addAll(OkHttpHooker.globalNetworkInterceptors);
}

```


这是OkhttpClient中内部类Builder的构造函数，我们的目标是在方法末尾加上四行代码，这样一来，所有的OkHttpClient都会拥有共同的Intercepter/Dns/EventListener。我们再来看看OkHttpHooker的实现

```java

public class OkHttpHooker {

    public static EventListener.Factory globalEventFactory = new EventListener.Factory() {
        public EventListener create(Call call) {
            return EventListener.NONE;
        }
    };;

    public static Dns globalDns = Dns.SYSTEM;

    public static List<Interceptor> globalInterceptors = new ArrayList<>();

    public static List<Interceptor> globalNetworkInterceptors = new ArrayList<>();

    public static void installEventListenerFactory(EventListener.Factory factory) {
        globalEventFactory = factory;
    }

    public static void installDns(Dns dns) {
        globalDns = dns;
    }

    public static void installInterceptor(Interceptor interceptor) {
        if(interceptor != null)
            globalInterceptors.add(interceptor);
    }

    public static void installNetworkInterceptors(Interceptor networkInterceptor) {
        if(networkInterceptor != null)
            globalNetworkInterceptors.add(networkInterceptor);
    }


}
```

这样，只需要为OkHttpHooker预先install好几个全局的Intercepter/Dns/EventListener即可。


那么，如何来实现上面OkhttpClient内部Builder中插入四行代码呢？

首先，我们通过Hunter的框架，可以隐藏掉Transform和ASM绝大部分细节，我们只需把注意力放在写ClassVisitor以及MethodVisitor即可。我们一共需要做以下几步

1、新建一个自定义transform，添加到一个自定义gradle plugin中
2、继承HunterTransform实现自定义transform
3、实现自定义的ClassVisitor，并依情况实现自定义MethodVisitor


其中第一步文章讲解transform一部分有讲到，基本是一样简短的写法，我们从第二步讲起

继承HunterTransform，就可以让你的transform具备并发、增量的功能。

```java

final class OkHttpHunterTransform extends HunterTransform {

    private Project project;
    private OkHttpHunterExtension okHttpHunterExtension;

    public OkHttpHunterTransform(Project project) {
        super(project);
        this.project = project;
        //依情况而定，看看你需不需要有插件扩展
        project.getExtensions().create("okHttpHunterExt", OkHttpHunterExtension.class);
        //必须的一步，继承BaseWeaver，帮你隐藏ASM细节
        this.bytecodeWeaver = new OkHttpWeaver();
    }

    @Override
    public void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        okHttpHunterExtension = (OkHttpHunterExtension) project.getExtensions().getByName("okHttpHunterExt");
        super.transform(context, inputs, referencedInputs, outputProvider, isIncremental);
    }

    // 用于控制修改字节码在哪些debug包还是release包下发挥作用，或者完全打开/关闭
    @Override
    protected RunVariant getRunVariant() {
        return okHttpHunterExtension.runVariant;
    }

}

//BaseWeaver帮你隐藏了ASM的很多复杂逻辑
public final class OkHttpWeaver extends BaseWeaver {

    @Override
    protected ClassVisitor wrapClassWriter(ClassWriter classWriter) {
        return new OkHttpClassAdapter(classWriter);
    }

}




//插件扩展
public class OkHttpHunterExtension {

    public RunVariant runVariant = RunVariant.ALWAYS;

    @Override
    public String toString() {
        return "OkHttpHunterExtension{" +
                "runVariant=" + runVariant +
                '}';
    }
}


```

好了，Transform写起来就变得这么简单，接下来看自定义ClassVisitor，它在OkHttpWeaver返回。


我们新建一个ClassVisitor(自定义ClassVisitor是为了代理ClassWriter，前面讲过)


```java
public final class OkHttpClassAdapter extends ClassVisitor{

    private String className;

    OkHttpClassAdapter(final ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
        this.className = name;
    }

    @Override
    public MethodVisitor visitMethod(final int access, final String name,
                                     final String desc, final String signature, final String[] exceptions) {
        MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);
        if(className.equals("okhttp3/OkHttpClient$Builder")) {
            return mv == null ? null : new OkHttpMethodAdapter(className + File.separator + name, access, desc, mv);
        } else {
            return mv;
        }
    }

}
```


我们寻找出`okhttp3/OkHttpClient$Builder`这个类，其他类不管它，那么其他类只会被普通的复制，而`okhttp3/OkHttpClient$Builder`将会有自定义的MethodVisitor来处理

我们来看看这个MethodVisitor的实现

```java

public final class OkHttpMethodAdapter extends LocalVariablesSorter implements Opcodes {

    private boolean defaultOkhttpClientBuilderInitMethod = false;

    OkHttpMethodAdapter(String name, int access, String desc, MethodVisitor mv) {
        super(Opcodes.ASM5, access, desc, mv);
        if ("okhttp3/OkHttpClient$Builder/<init>".equals(name) && "()V".equals(desc)) {
            defaultOkhttpClientBuilderInitMethod = true;
        }
    }

    @Override
    public void visitInsn(int opcode) {
        if(defaultOkhttpClientBuilderInitMethod) {
            if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {

                //EventListenFactory
                mv.visitVarInsn(ALOAD, 0);
                mv.visitFieldInsn(GETSTATIC, "com/hunter/library/okhttp/OkHttpHooker", "globalEventFactory", "Lokhttp3/EventListener$Factory;");
                mv.visitFieldInsn(PUTFIELD, "okhttp3/OkHttpClient$Builder", "eventListenerFactory", "Lokhttp3/EventListener$Factory;");

                //Dns
                mv.visitVarInsn(ALOAD, 0);
                mv.visitFieldInsn(GETSTATIC, "com/hunter/library/okhttp/OkHttpHooker", "globalDns", "Lokhttp3/Dns;");
                mv.visitFieldInsn(PUTFIELD, "okhttp3/OkHttpClient$Builder", "dns", "Lokhttp3/Dns;");

                //Interceptor
                mv.visitVarInsn(ALOAD, 0);
                mv.visitFieldInsn(GETFIELD, "okhttp3/OkHttpClient$Builder", "interceptors", "Ljava/util/List;");
                mv.visitFieldInsn(GETSTATIC, "com/hunter/library/okhttp/OkHttpHooker", "globalInterceptors", "Ljava/util/List;");
                mv.visitMethodInsn(INVOKEINTERFACE, "java/util/List", "addAll", "(Ljava/util/Collection;)Z", true);
                mv.visitInsn(POP);

                //NetworkInterceptor
                mv.visitVarInsn(ALOAD, 0);
                mv.visitFieldInsn(GETFIELD, "okhttp3/OkHttpClient$Builder", "networkInterceptors", "Ljava/util/List;");
                mv.visitFieldInsn(GETSTATIC, "com/hunter/library/okhttp/OkHttpHooker", "globalNetworkInterceptors", "Ljava/util/List;");
                mv.visitMethodInsn(INVOKEINTERFACE, "java/util/List", "addAll", "(Ljava/util/Collection;)Z", true);
                mv.visitInsn(POP);
            }
        }
        super.visitInsn(opcode);
    }

}

```

首先，我们先找出`okhttp3/OkHttpClient$Builder`的构造函数，然后在这个构造函数的末尾，执行插入字节码的逻辑，我们可以发现，字节码的指令是符合逆波兰式的，都是操作数在前，操作符在后。

至此，我们只需要发布插件，然后apply到我们的项目中即可。

借助Hunter框架，我们很轻松就成功hack了Okhttp，我们就可以用全局统一的Intercepter/Dns/EventListener来监控我们APP的网络了。另外，说个题外话，可能有人可能会说，并非所有网络请求都是通过Okhttp来写，也可能是通过URLConnection来写，可以参考我另一篇文章提到的 

http://quinnchen.me/2017/11/18/2017-11-18-android-http-dns/#HttpURLConnection

这篇文章有讲解如何将URLConnection的请求导向自己指定的OkhttpClient.

讲到这里，就完整得介绍了如何使用Hunter框架开发一个字节码编译插件，对第三方依赖库为所欲为。如果对于代码还有疑惑，可以移步项目主页，参考完整代码，以及其他几个插件的实现。有时间再写文章介绍其他几个插件的具体实现。



## 总结

这篇文章写到这里差不多了，全文主要围绕Hunter展开介绍，分析了如何开发一个高效的修改字节码的编译插件，以及ASM字节码技术的一些相关工作流和开发套路。

也还原大家前往[Hunter](https://github.com/Leaking/Hunter)项目主页，欢迎使用[Hunter](https://github.com/Leaking/Hunter)框架开发插件，以及使用现有的几个插件，也欢迎提issue，欢迎star/fork。

## 参考资料与文章推荐

[深入理解Java虚拟机](https://book.douban.com/subject/24722612/)

[ASM GUIDE](https://asm.ow2.io/asm4-guide.pdf)

[一份不错的字节码技术PPT](https://s3-eu-west-1.amazonaws.com/presentations2012/30_presentation.pdf)






