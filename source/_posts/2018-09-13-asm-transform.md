---
title: 如何使用ASM对你Android项目中的class为所欲为
date: 2018-09-13 23:12:06
tags: Android, JVM
---

作为Android开发，

是否也曾这样想，要对全局所有class插桩，做UI，内存，网络等等方面性能监控;
是否也曾这样想，某个第三方依赖，用得不爽，但是不想拿它的源码修改再重新编译，而想对它的class直接做点手脚，比如给Okhttp加一个全局的Interceptor，监控流量？
是否也曾这样想，每次写打log时，想让TAG自动生成，让它默认就是当前类的名称，甚至你想让log里自动加上当前代码所在的行数，当同个class中有多行相同日志时，才容易定位日志;
是否也曾这样想，Java自带的动态代理太弱了，只能对接口类做动态代理，而我们想对任何类做动态代理;

为了实现上面这些猜想，可能我们最开始的第一反应，都是能否通过代码生成技术来实现，或者反射、或者动态代理来实现，但是想来想去，APT什么的，貌似不能满足上面的需求，而且，以上这些问题都不能从Java文件入手，而应该从class文件寻找突破。而从class文件入手，我们就不得不来近距离接触一下字节码！

JVM平台上，修改、生成字节码无处不在，从ORM框架（如Hibernate, MyBatis）到Mock框架（如Mockio），再到Java Web中长盛不衰的Spring框架，再到新兴的JVM语言[Kotlin的编译器](https://github.com/JetBrains/kotlin/tree/v1.2.30/compiler/backend/src/org/jetbrains/kotlin/codegen)，还有大名鼎鼎的[cglib](https://github.com/cglib/cglib)项目，都有字节码的身影。

字节码相关技术的强大之处不用多说，而且Android开发中，无论是使用Java开发和Kotlin开发，都是JVM平台的语言，所以如果我们在Android开发中，使用字节码技术做一下hack，还可以天然地兼容Java和Kotlin语言，真香。

这篇文章将介绍字节码技术在Android开发中的应用，主要围绕以下几点展开：

+ Transform的原理与应用
+ 字节码基础知识
+ ASM的介绍与如何应用到gradle插件中
+ 几个具体应用案例

话不多说，让我们开始吧


## 一、Transform的原理与应用

我们在这里先引出一个概念，就是Android gradle plugin 1.5开始引入的[Transform](http://google.github.io/android-gradle-dsl/javadoc/3.2/)。接下来来让我们一起来好好研究下Transform。

我们先从如何引入Transform依赖说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中，让我们看看Transform在[jcenter](https://dl.bintray.com/android/android-tools/com/android/tools/build/transform-api/)上的历个版本。

![](/images/transform_2.png)

所以，很久很久以前我引入transform依赖是这样

```groovy
compile 'com.android.tools.build:transform-api:1.5.0'

```

现在是这样

```groovy
implementation 'com.android.tools.build:gradle-api:3.1.4'  //从2.0.0版本开始就是在gradle-api中了

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

接下来介绍Transform，介绍怎么用它之前，先介绍一下它的原理，一图胜千言，请看图


![](/images/transformconsume_transform.png)



从图中可知，每个Transform其实都是一个gradle task，Android编译器将每个Transform串连起来，第一个Transform接收来自javac编译class的结果，以及已经拉取到在本地的第三方依赖（jar. aar），还有resource资源，注意，这里的resource并非res资源，而是asset目录下的资源。这些编译的中间产物，在Transform组成的链条上流动，每个Transform节点可以对class进行处理再传递给下一个Transform。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个Transform中，而我们自定义的Transform，会插入到这个Transform链条的最前面。


但其实，上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。至于怎么在一个Transform中声明两种输入，以及怎么处理两种输入，后面将有示例代码。


为了印证Transform的工作原理和应用方式，我们也可以从Android gradle plugin源码入手找出证据，在TaskManager中，有一个方法`createPostCompilationTasks`.


```java

/**
 * Creates the post-compilation tasks for the given Variant.
 *
 * These tasks create the dex file from the .class files, plus optional intermediary steps like
 * proguard and jacoco
 */
public void createPostCompilationTasks(

        @NonNull final VariantScope variantScope) {

    checkNotNull(variantScope.getJavacTask());

    final BaseVariantData variantData = variantScope.getVariantData();
    final GradleVariantConfiguration config = variantData.getVariantConfiguration();

    TransformManager transformManager = variantScope.getTransformManager();

    // ---- Code Coverage first -----
    boolean isTestCoverageEnabled =
            config.getBuildType().isTestCoverageEnabled()
                    && !config.getType().isForTesting()
                    && !variantScope.getInstantRunBuildContext().isInInstantRunMode();
    if (isTestCoverageEnabled) {
        createJacocoTransform(variantScope);
    }

    maybeCreateDesugarTask(variantScope, config.getMinSdkVersion(), transformManager);

    AndroidConfig extension = variantScope.getGlobalScope().getExtension();

    // Merge Java Resources.
    createMergeJavaResTransform(variantScope);

    // ----- External Transforms -----
    // apply all the external transforms.
    List<Transform> customTransforms = extension.getTransforms();
    List<List<Object>> customTransformsDependencies = extension.getTransformsDependencies();

    for (int i = 0, count = customTransforms.size(); i < count; i++) {
        Transform transform = customTransforms.get(i);

        List<Object> deps = customTransformsDependencies.get(i);
        transformManager
                .addTransform(taskFactory, variantScope, transform)
                .ifPresent(
                        t -> {
                            if (!deps.isEmpty()) {
                                t.dependsOn(deps);
                            }

                            // if the task is a no-op then we make assemble task depend on it.
                            if (transform.getScopes().isEmpty()) {
                                variantScope.getAssembleTask().dependsOn(t);
                            }
                        });
    }

    // ----- Android studio profiling transforms
    for (String jar : getAdvancedProfilingTransforms(projectOptions)) {
        if (variantScope.getVariantConfiguration().getBuildType().isDebuggable()
                && variantData.getType().equals(VariantType.DEFAULT)
                && jar != null) {
            transformManager.addTransform(
                    taskFactory, variantScope, new CustomClassTransform(jar));
        }
    }

    // ----- Minify next -----
    maybeCreateJavaCodeShrinkerTransform(variantScope);

    maybeCreateResourcesShrinkerTransform(variantScope);

    // ----- 10x support

    PreColdSwapTask preColdSwapTask = null;
    if (variantScope.getInstantRunBuildContext().isInInstantRunMode()) {

        DefaultTask allActionsAnchorTask = createInstantRunAllActionsTasks(variantScope);
        assert variantScope.getInstantRunTaskManager() != null;
        preColdSwapTask =
                variantScope.getInstantRunTaskManager().createPreColdswapTask(projectOptions);
        preColdSwapTask.dependsOn(allActionsAnchorTask);

        // force pre-dexing to be true as we rely on individual slices to be packaged
        // separately.
        extension.getDexOptions().setPreDexLibraries(true);
        variantScope.getInstantRunTaskManager().createSlicerTask();

        extension.getDexOptions().setJumboMode(true);
    }
    // ----- Multi-Dex support

    DexingType dexingType = variantScope.getDexingType();

    // Upgrade from legacy multi-dex to native multi-dex if possible when using with a device
    if (dexingType == DexingType.LEGACY_MULTIDEX) {
        if (variantScope.getVariantConfiguration().isMultiDexEnabled()
                && variantScope
                                .getVariantConfiguration()
                                .getMinSdkVersionWithTargetDeviceApi()
                                .getFeatureLevel()
                        >= 21) {
            dexingType = DexingType.NATIVE_MULTIDEX;
        }
    }

    Optional<TransformTask> multiDexClassListTask;

    if (dexingType == DexingType.LEGACY_MULTIDEX) {
        boolean proguardInPipeline = variantScope.getCodeShrinker() == CodeShrinker.PROGUARD;

        // If ProGuard will be used, we'll end up with a "fat" jar anyway. If we're using the
        // new dexing pipeline, we'll use the new MainDexListTransform below, so there's no need
        // for merging all classes into a single jar.
        if (!proguardInPipeline && !usingIncrementalDexing(variantScope)) {
            // Create a transform to jar the inputs into a single jar. Merge the classes only,
            // no need to package the resources since they are not used during the computation.
            JarMergingTransform jarMergingTransform =
                    new JarMergingTransform(TransformManager.SCOPE_FULL_PROJECT);
            transformManager
                    .addTransform(taskFactory, variantScope, jarMergingTransform)
                    .ifPresent(variantScope::addColdSwapBuildTask);
        }

        // ---------
        // create the transform that's going to take the code and the proguard keep list
        // from above and compute the main class list.
        Transform multiDexTransform;
        if (usingIncrementalDexing(variantScope)) {
            if (projectOptions.get(BooleanOption.ENABLE_D8_MAIN_DEX_LIST)) {
                multiDexTransform = new D8MainDexListTransform(variantScope);
            } else {
                multiDexTransform =
                        new MainDexListTransform(variantScope, extension.getDexOptions());
            }
        } else {
            multiDexTransform = new MultiDexTransform(variantScope, extension.getDexOptions());
        }
        multiDexClassListTask =
                transformManager.addTransform(taskFactory, variantScope, multiDexTransform);
        multiDexClassListTask.ifPresent(variantScope::addColdSwapBuildTask);
    } else {
        multiDexClassListTask = Optional.empty();
    }


    if (usingIncrementalDexing(variantScope)) {
        createNewDexTasks(variantScope, multiDexClassListTask.orElse(null), dexingType);
    } else {
        createDexTasks(variantScope, multiDexClassListTask.orElse(null), dexingType);
    }

    if (preColdSwapTask != null) {
        for (DefaultTask task : variantScope.getColdSwapBuildTasks()) {
            task.dependsOn(preColdSwapTask);
        }
    }

    // ---- Create tasks to publish the pipeline output as needed.

    final File intermediatesDir = variantScope.getGlobalScope().getIntermediatesDir();
    createPipelineToPublishTask(
            variantScope,
            transformManager.getPipelineOutputAsFileCollection(StreamFilter.DEX),
            FileUtils.join(intermediatesDir, "bundling", "dex"),
            PUBLISHED_DEX);

    createPipelineToPublishTask(
            variantScope,
            transformManager.getPipelineOutputAsFileCollection(StreamFilter.RESOURCES),
            FileUtils.join(intermediatesDir, "bundling", "java-res"),
            PUBLISHED_JAVA_RES);

    createPipelineToPublishTask(
            variantScope,
            transformManager.getPipelineOutputAsFileCollection(StreamFilter.NATIVE_LIBS),
            FileUtils.join(intermediatesDir, "bundling", "native-libs"),
            PUBLISHED_NATIVE_LIBS);
}

```

这个方法的脉络很清晰，我们可以看到，Jacoco，Desugar，MergeJavaRes，AdvancedProfiling，Shrinker，Proguard, JarMergeTransform, MultiDex, Dex都是通过Transform的形式一个个串联起来。其中也有将我们自定义的Transform插进去。


讲完了Transform的数据流动的原理，再介绍Transform的输入数据的过滤机制，Transform的数据输入，可以通过Scope和ContentType两个维度进行过滤。

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

可以看到，在transform方法中，我们将每个jar包和class文件复制到dest路径，这个dest路径就是下一个Transform的输入数据，而在复制时，我们就可以做一些狸猫换太子，偷天换日的事情了，先将jar包和class文件的字节码做一些修改，再进行复制即可，至于怎么修改字节码，就要借助我们后面介绍的ASM了。而如果开发过程要看你当前transform处理之后的class/jar包，可以到/build/intermediates/transforms/CustomTransform/下查看，你会发现所有jar包命名都是123456递增，这是正常的，这里的命名规则可以在OutputProvider.getContentLocation的具体实现中找到

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



到此为止，看起来Transform用起来也不难，但是，如果直接这样使用，会大大拖慢编译时间，为了解决这个问题，摸索了一段时间后，也借鉴了Android编译器中Desugar等几个Transform的实现，发现我们可以使用增量编译，并且上面transform方法遍历处理每个jar/class的流程，其实可以并发处理，加上一般编译流程都是在PC上，所以我们可以尽量敲诈机器的资源。

想要开启增量编译，我们需要重写Transform的这个接口，返回true。


```java
@Override 
public boolean isIncremental() {
    return true;
}

```


开启了支持增量编译后，我们可以在transform方法中，检查当前编译是否是增量编译。

如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理
如果是增量编译，则要检查每个文件的Status，Status分四种，

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


实现了增量编译后，我们最好也支持并发编译，并发编译的实现并不乏咋，只需要将上面处理单个jar/class的逻辑，并发处理，最后阻塞等待所有任务结束即可。

后面做各种模式下的编译速度对比，会发现增量和并发对编译速度的影响是很显著的，而我在查看Android gradle plugin自身的十几个Transform时，发现它们实现方式也有一些区别，有些用kotlin写，有些用java写，有些支持增量，有些不支持，而且是代码注释写了一个大大的FIXME, To support incremental build。所以，讲道理，现阶段的Android编译速度，还是有提升空间的。


上面我们介绍了Transform，以及如何高效地在编译期间寻找时机处理所有字节码，那么具体怎么处理字节码呢？接下来来让我们一起来看看JVM平台上的字节码神兵利器，ASM!


## 二、ASM

ASM的官网在这里[https://asm.ow2.io/](https://asm.ow2.io/)，随便贴一下它的主页介绍，一起感受下它的强大


![](/images/transform_asm_introduce.png)


JVM平台上，处理字节码的框架最常见的就三个，ASM，Javasist，AspectJ。我尝试过Javasist，而AspectJ也稍有了解，最终选择ASM，因为使用它可以更底层地处理字节码的每条命令，处理速度、内存占用，也是优于其他两个框架。






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






