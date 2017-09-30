---
title: AndroidPlugin源码解析(一)
date: 2017-05-30
tags: 
---

## 前言

上一篇我们知道`com.android.application` plugin对应的类为`com.android.build.gradle.AppPlugin`

这个类的声明为: `public class AppPlugin extends BasePlugin implements Plugin<Project>`

看到最后`implements Plugin<Project>`，熟悉gradle plugin的人应该知道， 
`apply plugin: 'com.android.application'`其实就是调用`AppPlugin类`的`public void apply(@NonNull Project project)`方法。

`com.android.library` plugin对应的类为`com.android.build.gradle.LibraryPlugin`。

这个类的声明为: `public class LibraryPlugin extends BasePlugin implements Plugin<Project>`

这个两个`apply`方法的具体内容只有一点点不同，两个都调用了`super.apply(project)`方法:

~~~ java
@Override
public void apply(@NonNull Project project) {
    super.apply(project);
    // 下面的代码只在Library plugin中存在
    // Default assemble task for the default-published artifact.
    // This is needed for the prepare task on the consuming project.
    project.getTasks().create("assembleDefault");
}
~~~ 

接下来我们一步步的跟进:

### BasePlugin apply

可以看到，就是调用了BasePlugin的`apply`方法。我们来看一下BasePlugin的`apply`方法做了什么。
老规矩，分析中会去掉日志，记录，错误检查之类的代码。

~~~ java
// 去掉那些我认为不重要的代码后，整个apply方法的流程如下。
protected void apply(@NonNull Project project) {
    configureProject();
    createExtension();
    createTasks();
    // 读取 "android.additional.plugins"对应的插件名称
    // Apply additional plugins
    for (String plugin : AndroidGradleOptions.getAdditionalPlugins(project)) {
        project.apply(ImmutableMap.of("plugin", plugin));
    }
}
~~~

可以看到，其实整个`apply`方法分4步，最后一步读取插件没有什么好分析的，对于剩下三步来说本篇文章只分析前两步。

### BasePlugin configureProject

我们一个过程一个过程研究。
先看`configureProject()`

~~~ java
String creator = "Android Gradle " Version.ANDROID_GRADLE_PLUGIN_VERSION;

protected void configureProject() {
    sdkHandler = new SdkHandler(project, getLogger());
    
    androidBuilder = new AndroidBuilder(
            project == project.getRootProject() ? project.getName() : project.getPath(), // project Id
            creator, // the createdBy String for the apk manifest（打包之后的MANIFEST.MF文件可以看到）
            new GradleProcessExecutor(project),  // An executor for external processes.
            new GradleJavaProcessExecutor(project), // An executor for external Java-based processes.
            extraModelInfo,
            getLogger(),
            isVerbose());
    
    project.getPlugins().apply(JavaBasePlugin.class);
    jacocoPlugin = project.getPlugins().apply(JacocoPlugin.class);
    project.getTasks().getByName("assemble").setDescription(
            "Assembles all variants of all applications and secondary packages.");
    // call back on execution. This is called after the whole build is done (not
    // after the current project is done).
    // This is will be called for each (android) projects though, so this should support
    // being called 2+ times.
    project.getGradle().addBuildListener(new BuildListener() {
        private final LibraryCache libraryCache = LibraryCache.getCache();
        @Override
        public void buildStarted(Gradle gradle) { }
        @Override
        public void settingsEvaluated(Settings settings) { }
        @Override
        public void projectsLoaded(Gradle gradle) { }
        @Override
        public void projectsEvaluated(Gradle gradle) { }
        @Override
        public void buildFinished(BuildResult buildResult) {
            ExecutorSingleton.shutdown();
            sdkHandler.unload();
            PreDexCache.getCache().clear(
                                    new File(project.getRootProject().getBuildDir(),
                                    FD_INTERMEDIATES + "/dex-cache/cache.xml"),
                                    getLogger());
                            JackConversionCache.getCache().clear(
                                    new File(project.getRootProject().getBuildDir(),
                                    FD_INTERMEDIATES + "/jack-cache/cache.xml"),
                                    getLogger());
            libraryCache.unload();
            Main.clearInternTables();
        }
    });
    project.getGradle().getTaskGraph().addTaskExecutionGraphListener(
            new TaskExecutionGraphListener() {
                @Override
                public void graphPopulated(TaskExecutionGraph taskGraph) {
                    for (Task task : taskGraph.getAllTasks()) {
                        if (task instanceof TransformTask) {
                            Transform transform = ((TransformTask) task).getTransform();
                            if (transform instanceof DexTransform) {
                                PreDexCache.getCache().load(
                                        new File(project.getRootProject().getBuildDir(),
                                        FD_INTERMEDIATES + "/dex-cache/cache.xml"));
                                break;
                            } else if (transform instanceof JackPreDexTransform) {
                                JackConversionCache.getCache().load(
                                        new File(project.getRootProject().getBuildDir(),
                                        FD_INTERMEDIATES + "/jack-cache/cache.xml"));
                                break;
                            }
                        }
                    }
                }
            });
}       
~~~

这个方法主要做了如下几个操作:
- 初始化SdkHandler
- 初始化AndroidBuilder
- 依赖JavaBasePlugin，JacocoPlugin；这两个Plugin的apply()方法我这里就不写了，大家有兴趣就像本文一样跟进去即可。
- 给"assemble"任务加上说明
- 当Gradle Build完成的时候做一些关闭和清理的操作
- 在Transform任务真正执行之前，读取对应的Cache。

SdkHandler主要作用是找local.properties文件里SdkLocation和NdkLocation
SdkLocation的查找顺序为：
    1. local.properties文件里sdk.dir的配置
    2. local.properties文件里android.dir的配置
    3. 环境变量"ANDROID_HOME"
    4. 环境变量"android.home"

NdkLocation的查找顺序为：
    1. local.properties文件里ndk.dir的配置
    2. 环境变量"ANDROID_NDK_HOME"


AndroidBuilder对象主要的几个参数的含义从我在代码中添加的注释即可了解。
最后讲一下`TaskExecutionGraphListener`这个接口。
官方文档是这样说的: 

>
    A TaskExecutionGraphListener is notified when the TaskExecutionGraph has been populated. 
    You can use this interface in your build file to perform some action based on the contents of the graph, 
    before any tasks are actually executed.

大概翻译一下，当Gradle将所有task的关系图填充之后，该接口会被调用。
你可以在build文件中使用这个接口，在任意task真正执行之前，去完成一些基于task关系图内容的任务。
所以这里可以根据对应的Task，判断Task的类型，做对应的操作。


### BasePlugin createExtension

继续看`apply`方法中第二个过程`createExtension()`

~~~ java
private void createExtension() {
    // 保存BuildType的NamedDomainObjectContainer
    final NamedDomainObjectContainer<BuildType> buildTypeContainer = project.container(
            BuildType.class,
            new BuildTypeFactory(instantiator, project, project.getLogger())); 
    // 保存ProductFlavor的NamedDomainObjectContainer
    final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer = project.container(
            ProductFlavor.class,
            new ProductFlavorFactory(instantiator, project, project.getLogger(), extraModelInfo));
    // 保存SigningConfig的NamedDomainObjectContainer
    final NamedDomainObjectContainer<SigningConfig>  signingConfigContainer = project.container(
            SigningConfig.class,
            new SigningConfigFactory(instantiator));
    // 创建android这个extension
    extension = project.getExtensions().create("android", getExtensionClass(),
            project, instantiator, androidBuilder, sdkHandler,
            buildTypeContainer, productFlavorContainer, signingConfigContainer,
            extraModelInfo, isLibrary());

    // create the default mapping configuration.
    // 创建default-mapping和default-metadata两个configuration，并加入描述
    // 具体的作用暂时没有看出来,如果你知道，欢迎留言告诉我
    // 通过如下代码放在build.gradle，可以打印出当前所有的configuration
    /**
        configurations.names.forEach(new Consumer<String>() {
                @Override
                void accept(String s) {
                        println "configurations：" + s;
                }
        })
    */
    project.getConfigurations().create("default" + VariantDependencies.CONFIGURATION_MAPPING)
            .setDescription("Configuration for default mapping artifacts.");
    project.getConfigurations().create("default" + VariantDependencies.CONFIGURATION_METADATA)
            .setDescription("Metadata for the produced APKs.");
    // 创建DependencyManager, 负责管理配置的所有依赖
    dependencyManager = new DependencyManager(
            project,
            extraModelInfo,
            sdkHandler);
    // 负责管理ndk相关，这里略过不提
    ndkHandler = new NdkHandler(
            project.getRootDir(),
            null, /* compileSkdVersion, this will be set in afterEvaluate */
            "gcc",/* toolchainName */
            "" /*toolchainVersion*/);
    // 负责创建taskManager
    taskManager = createTaskManager(
            project,
            androidBuilder,
            dataBindingBuilder,
            extension,
            sdkHandler,
            ndkHandler,
            dependencyManager,
            registry);
    
    // 负责创建variant
    variantFactory = createVariantFactory();
    variantManager = new VariantManager(
            project,
            androidBuilder,
            extension,
            variantFactory,
            taskManager,
            instantiator);

    // Register a builder for the custom tooling model
    ModelBuilder modelBuilder = new ModelBuilder(
            androidBuilder,
            variantManager,
            taskManager,
            extension,
            extraModelInfo,
            ndkHandler,
            new NativeLibraryFactoryImpl(ndkHandler),
            isLibrary(),
            AndroidProject.GENERATION_ORIGINAL);
    registry.register(modelBuilder);

    // Register a builder for the native tooling model
    NativeModelBuilder nativeModelBuilder = new NativeModelBuilder(variantManager);
    registry.register(nativeModelBuilder);

    // 下面三个whenObjectAdded方法从名字上就很好理解
    // 当你添加一个对应的对象的时候，回调此方法，也就是需要在variantManager里面添加对应的对象

    // map the whenObjectAdded callbacks on the containers.
    signingConfigContainer.whenObjectAdded(new Action<SigningConfig>() {
        @Override
        public void execute(SigningConfig signingConfig) {
            variantManager.addSigningConfig(signingConfig);
        }
    });

    buildTypeContainer.whenObjectAdded(new Action<BuildType>() {
        @Override
        public void execute(BuildType buildType) {
            // 默认debug签名
            SigningConfig signingConfig = signingConfigContainer.findByName(BuilderConstants.DEBUG);
            buildType.init(signingConfig);
            variantManager.addBuildType(buildType);
        }
    });

    productFlavorContainer.whenObjectAdded(new Action<ProductFlavor>() {
        @Override
        public void execute(ProductFlavor productFlavor) {
            variantManager.addProductFlavor(productFlavor);
        }
    });

    // 同上面类似，三个回调方法，不过是移除的时候抛出异常
    // 这个只能讲一下注释，具体如何复现暂时我不了解
    // map whenObjectRemoved on the containers to throw an exception.
    signingConfigContainer.whenObjectRemoved(
            new UnsupportedAction("Removing signingConfigs is not supported."));
    buildTypeContainer.whenObjectRemoved(
            new UnsupportedAction("Removing build types is not supported."));
    productFlavorContainer.whenObjectRemoved(
            new UnsupportedAction("Removing product flavors is not supported."));
    
    // 最后创建了这些默认的对象
    // create default Objects, signingConfig first as its used by the BuildTypes.
    variantFactory.createDefaultComponents(
            buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
~~~

看到这么长的代码，本想去掉点，发现这里面去什么都不合适。
那么跟着我慢慢看一下。这里有几个地方需要解释:

- `NamedDomainObjectContainer`是什么？

`NamedDomainObjectContainer`简单来说就是一个容器，里面的泛型<T>保存了容器里面需要传的类型，类似一个List<T>。

- extension的创建代表了什么?

extension的创建代表了我们在build.gradle文件中，可以使用

~~~ gradle
android{

}
~~~

这样的闭包来给该扩展传消息。我们在`{}`内写的任何被允许的内容，在这里都被定义好了。从`getExtensionClass`方法获取具体的Class，
application: 使用`com.android.build.gradle.AppExtension.class`
library: 使用`com.android.build.gradle.LibraryExtension.class`。
后面的值都是负责初始化这两个类的时候传入的值。

- TaskManager的作用?

首先这个依旧根据application和library做区分。
application: 创建`ApplicationTaskManager`
library: 创建`LibraryTaskManager`
这两个类就是负责创建我们用来处理Android源码的Gradle Task的地方，比如assemblyDebug, installDebug...
现在这里大概了解一下这个对象，后面再细细解读此对象。

- VariantManager的作用？

这就是管理`Build Variant`的地方，`Build Variant`的意义就是` build type`和`product flavor`交叉形成的，换句话说就是合并这两个东东的属性形成`Build Variant`的配置。
暂时也是单纯的提一下，后面再细细解读。

`ModelBuilder`, `NativeModelBuilder`和`private ToolingModelBuilderRegistry registry`这三个对象我们都先放一下，
这个和Android Plugin本身无关，是和gradle的运行机制相关的东东。

好了，这里大概梳理一下此方法的逻辑，
- 创建`buildTypeContainer`, `productFlavorContainer`, `signingConfigContainer`
- 将android这个extension加入build.gradle
- 创建一些后面要使用的对象主要指`ModelBuilder`和`NativeModelBuilder`,注册到gradle中。
- 给上面提到的Container加一些回调
- 通过`variantFactory`给Container创建默认的配置

看一下这个默认的配置，App和Library的默认配置是相同的，都是如下代码。


#### VariantFactory createDefaultComponents

~~~ java
@Override
public void createDefaultComponents(
        @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
        @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
        @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs) {
    // must create signing config first so that build type 'debug' can be initialized
    // with the debug signing config.
    signingConfigs.create(DEBUG);
    buildTypes.create(DEBUG);
    buildTypes.create(RELEASE);
}
~~~

可以看出来我们的SigningConfigs里面默认有debug类型。
为什么不需要写signingConfig就有个默认的debug签名？就是这里配置的。
同理，为什么新建工程buildTypes有debug和release？就是这里创建的。
这里就相当于我们在`build.gradle`文件写了如下内容:

~~~ java
android{
    signingConfigs {
        debug {}
    }

    buildTypes {
        debug {}
        release {}
    }
}
~~~

这个方法到这里也就暂时过完了。所以这一篇到这里就告一段落了。

