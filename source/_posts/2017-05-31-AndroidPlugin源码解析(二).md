---
title: AndroidPlugin源码解析(二)
date: 2017-05-31
tags:
---

## 前言

上一篇文章还没分析完，还留了剩下两步，这一篇我们继续跟进。
先从第三步开始：

### BasePlugin createTasks

看`apply`方法中第三个过程`createTasks()`

~~~ java
private void createTasks() {
    taskManager.createTasksBeforeEvaluate(
                            new TaskContainerAdaptor(project.getTasks()));
    createAndroidTasks(false);
}
~~~

方法很短，里面的内容就两个方法。
第一个方法里面创建了一些task，可是这些task和我们打包过程基本无关，所以这里忽视掉。
跳过了一些ndk相关的代码，来看看第二个方法

#### BasePlugin createAndroidTasks

~~~ java
@VisibleForTesting
final void createAndroidTasks(boolean force) {
    // 检测buildToolVersion,compileSdkVersion的设置
    // Make sure unit tests set the required fields.
    checkState(extension.getBuildToolsRevision() != null,
            "buildToolsVersion is not specified.");
    checkState(extension.getCompileSdkVersion() != null, "compileSdkVersion is not specified.");

    // 不允许依赖java plugin
    // get current plugins and look for the default Java plugin.
    if (project.getPlugins().hasPlugin(JavaPlugin.class)) {
        throw new BadPluginException(
                "The 'java' plugin has been applied, but it is not compatible with the Android plugins.");
    }
    
    // 初始化TargetInfo这个类
    // 这个类负责给build project的时候提供需要的信息
    ensureTargetSetup();

    // don't do anything if the project was not initialized.
    // Unless TEST_SDK_DIR is set in which case this is unit tests and we don't return.
    // This is because project don't get evaluated in the unit test setup.
    // See AppPluginDslTest
    if (!force
            && (!project.getState().getExecuted() || project.getState().getFailure() != null)
            && SdkHandler.sTestSdkFolder == null) {
        return;
    }

    if (hasCreatedTasks) {
        return;
    }
    hasCreatedTasks = true;

    extension.disableWrite();

    // 移除远端Android sdk Maven依赖，改为本地，并将Android sdk Maven依赖放置到最前方
    // setup Android SDK repositories.
    sdkHandler.addLocalRepositories(project);
    // databinding相关
    taskManager.addDataBindingDependenciesIfNecessary(extension.getDataBinding());
  
    variantManager.createAndroidTasks();
    
    ApiObjectFactory apiObjectFactory = new ApiObjectFactory(
            androidBuilder, extension, variantFactory, instantiator);
    for (BaseVariantData variantData : variantManager.getVariantDataList())  {
        apiObjectFactory.create(variantData);
    }
}
~~~

前面的通过注释能了解一个大概，后面的很重要，也无法用注释来简单说明。
这个方法里有一个方法是最重要的，那就是`variantManager.createAndroidTasks();`
我们来看一看这个方法

#### VariantManager createAndroidTasks

~~~ java
public void createAndroidTasks() {
    // 这里主要是检测Libray plugin情况下 ProductFlavor和BuildType不能添加applicationIdSuffix和Jack支持
    // 以及ProductFlavor不能重写applicationId
    variantFactory.validateModel(this);
    // 这个只是在AndroidTest的时候做了操作
    variantFactory.preVariantWork(project);

    final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());
    // 生成我们选择最终编译出来的Variant, 比如debugProd。就是BuildType和ProductFlaovr结合生成的
    // ProductFlavor自身也可以通过dimensions结合，参见http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Multi-flavor-variants
    if (variantDataList.isEmpty()) {
        populateVariantDataList();
    }
    // Create top level test tasks.
    taskManager.createTopLevelTestTasks(tasks, !productFlavors.isEmpty());

    for (final BaseVariantData<? extends BaseVariantOutputData> variantData : variantDataList) {
        createTasksForVariantData(tasks, variantData);
    }
    // 类似androidDependencies之类的获取信息的task
    taskManager.createReportTasks(tasks, variantDataList);
}
~~~

这里提一点，我们项目里出现的`defaultConfig`其实就是`ProductFlavor`；
所以，所有`defaultConfig`里出现的设置，都可以在`ProductFlavor`中修改。

我们来仔细看一下这个方法`createTasksForVariantData(tasks, variantData);`,这个方法描述了打包流程。所以这个方法很重要。
去掉为测试生成的task代码

#### VariantManager createTasksForVariantData

~~~ java
/**
 * Create tasks for the specified variantData.
 */
public void createTasksForVariantData(
        final TaskFactory tasks,
        final BaseVariantData<? extends BaseVariantOutputData> variantData) {
    final BuildTypeData buildTypeData = buildTypes.get(
            variantData.getVariantConfiguration().getBuildType().getName());
    // 创建类似于AssembleDebug,　AssembleRelease之类的Ｔａｓｋ
    if (buildTypeData.getAssembleTask() == null) {
        buildTypeData.setAssembleTask(taskManager.createAssembleTask(tasks, buildTypeData));
    }
    // Add dependency of assemble task on assemble build type task.
    tasks.named("assemble", new Action<Task>() {
        @Override
        public void execute(Task task) {
            assert buildTypeData.getAssembleTask() != null;
            task.dependsOn(buildTypeData.getAssembleTask().getName());
        }
    });

    VariantType variantType = variantData.getType();
    // 把ProductFlaovr的Assemble任务加上
    createAssembleTaskForVariantData(tasks, variantData);

    taskManager.createTasksForVariantData(tasks, variantData);
}
~~~

这个方法里最重要的就是最后一行`taskManager.createTasksForVariantData(tasks, variantData);`，不管Test的情况下分为两种实现，一种是application，一种是library，这里我们先放着，因为这里是整个打包过程具体实现的地方，下一系列文章(Android打包过程解析)来解析这个方法的代码。
本篇文章继续往下跟进。

打包过程的第一阶段——配置，到这里还剩最后一个方法就完成了，我们来看看。

~~~ java
ApiObjectFactory apiObjectFactory = new ApiObjectFactory(
    androidBuilder, extension, variantFactory, instantiator);
    // variantManager.getVariantDataList() 就是前面创建的ProductFlavor
for (BaseVariantData variantData : variantManager.getVariantDataList())  {
    apiObjectFactory.create(variantData);
}
~~~

我们来看看`create`方法

#### ApiObjectFactory create

~~~ java
public void create(BaseVariantData<?> variantData) {
    // 这个方法主要目的是把output对象和variant对象关联起来
    BaseVariant variantApi =
            variantFactory.createVariantApi(variantData, readOnlyObjectProvider);
    // Only add the variant API object to the domain object set once it's been fully
    // initialized.
    extension.addVariant(variantApi);
}
~~~

不管Test相关的代码的话，我们可以看到，这个方法变成了只有两行，且最后一个方法的注释明显的写明了初始化完毕(it's been fully initialized)。

最后那个extension对象就是我们刚开始用的Android Extension对象。将variant加入Android Extension对象，初始化就完毕了。