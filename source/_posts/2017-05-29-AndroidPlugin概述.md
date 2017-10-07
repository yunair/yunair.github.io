---
title: Android Plugin 概述
date: 2017-05-29
tags: 
---

## Plugin前言

用Android Studio的会发现，我们新建的Android工程会在根目录的`build.gradle`文件内引入一个classpath:
`classpath 'com.android.tools.build:gradle:x.y.z'`,

我们还会在`build.gradle`引入如下的plugin:

~~~ gradle
// app 
apply plugin: 'com.android.application'
// library
apply plugin: 'com.android.library'
~~~

我们同时会在如下的范围内配置各种东东

~~~ gradle
android{

}
~~~

这到底是怎么工作的呢？

## 简述

这其实是一个Gradle Plugin，它表示引入了`com.android.tools.build:gradle-x.y.z`的jar包。这个包需要在repositories里面声明的位置找。
引入了这样一个jar包，我们才能引入必要的plugin:

~~~ gradle
// app 
apply plugin: 'com.android.application'
// library
apply plugin: 'com.android.library'
~~~

我们`apply plugin`之后，如果对应的plugin接受配置，我们就可以按照plugin的要求进行配置。

~~~ gradle
android {
    // 以及这里面的一大堆配置
}
~~~

看了前面的内容，我们来分析一下`apply plugin`之后发生了什么。
基于`com.android.tools.build:gradle:2.2.2`分析。

在源码的gradle目录下，我们可以看到`resources/META-INF/gradle-plugins`目录下存在一些以.properties结尾的文件。
这些文件就是我们前面`apply plugin: xxx`的名字，每个配置文件里的内容说明引入该plugin相当于引入哪个具体的文件。
在这个系列的文章里，我们主要关注两个插件，
`com.android.application`和`com.android.library`

`com.android.application.properties`里的内容为:
`implementation-class=com.android.build.gradle.AppPlugin`

`com.android.library.properties`里的内容为:
`implementation-class=com.android.build.gradle.LibraryPlugin`

这个就指明了我们要研究的类；
application插件对应`com.android.build.gradle.AppPlugin`类
library插件对应`com.android.build.gradle.LibraryPlugin`类

好了，这篇简述就到此为止了，接下来分几篇来研究整个AndroidPlugin流程。