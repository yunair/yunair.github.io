---
title: Groovy & Gradle 入门
date: 2017-03-01
tags: 
---

## Groovy

**Groovy的API文档位于 http://www.groovy-lang.org/api.html**

### 字符串

1. 单引号''中的内容严格对应Java中的String，不对$符号进行转义

~~~ groovy
   def singleQuote='I am $ dolloar'  //输出就是I am $ dolloar
~~~

2.  双引号""的内容则和脚本语言的处理有点像，如果字符中有*$*符的话，则它会**$表达式**先求值。

~~~ groovy
   def doubleQuoteWithoutDollar = "I am one dollar" //输出 I am one dollar
   def x = 1
   def doubleQuoteWithDollar = "I am $x dolloar" //输出I am 1 dolloar
~~~


### 函数

调用的时候可以不加括号

~~~ groovy
println("test") ---->  println "test"
~~~

~~~ groovy
def getString(){
	//代码的最后一句是返回值
  "Hello World!"
  // 也可以使用return，和Java中普通函数一样
	// return "Hello World!"
}
~~~

### 闭包(Closure)

是一种数据类型，它代表了一段可执行的代码（类比Java的匿名内部类，可以被称为匿名函数）。其外形如下：

~~~ groovy
def closure = {//闭包是一段代码，所以需要用花括号括起来..
    String param1, int param2
    ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
    println "this is codes, $param1, $param2" //这是代码，最后一句是返回值
}
~~~

简而言之，Closure的定义格式是：

~~~ groovy
def closure = {params -> codes}  // or
def closure = {codes}  // no params ,so  no '->'
~~~

闭包定义好后，要调用它的方法就是：

~~~ groovy
closure.call(params) 
// 下面的方式也可以（groovy 的语法糖）
closure(params)
~~~

**如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫it，和this的作用类似。it代表闭包的参数。**

``` groovy
def greeting = { "Hello, $it!" }
assert greeting('XiaoKa') == 'Hello, XiaoKa!'
```



**Groovy中，调用函数时可以省略圆括号，当然，对于函数的参数里包含闭包同样适用。**

比如对于下面这个方法

~~~ groovy
public static <T> List<T> each(List<T> self, Closure closure)
~~~

~~~ groovy
// 就可以如下方式调用
def testList = [1,2,3,4,5]  //定义一个List
testList.each {
  println it
}
~~~

这个地方要记得，对于看gradle的代码很有帮助

~~~ groovy
doLast{
  println 'Hello World!'
}
~~~

完整代码应该是

~~~ groovy
doLast({
  println 'Hello World!'
})
~~~


## Gradle (基于3.2.1版本)

Gradle是什么:

**Gradle是一个框架，作为框架，它负责定义流程和规则。而具体的编译工作则是通过插件(Plugin)的方式来完成的。**


### Gradle工作流程


![img](/img/gradle/gradle.png)



可以看出，Gradle工作包含三个阶段：

- 首先是初始化阶段(Initiliazation phase):对Android项目而言，就是evaluate `settings.gradle`，并将每个Project内的build.gradle转化为Project对象。
- Configration阶段:配置对应的Project对象，`build.gradle`中有关build的代码将会被执行。Configuration阶段完了后，整个build的project以及内部的Task关系就确定了。Configuration会建立一个有向图来描述Task之间的依赖关系。
- 最后一个阶段就是执行阶段: 我们可以通过给`gradle`传递相应的命令来执行相应的task。


### 写Gradle代码

Gradle dsl api 文档: https://docs.gradle.org/current/dsl/

Gradle基于Groovy，Groovy又基于Java。所以，Gradle执行的时候和Groovy一样，会把脚本转换成Java对象。
Gradle主要有三种对象，这三种对象和三种不同的脚本文件对应，在gradle执行的时候，会将脚本转换成对应的对象：

- Gradle对象：当我们执行gradle xxx或者什么的时候，gradle会从默认的配置脚本中构造出一个Gradle对象。在整个执行过程中，只有这么一个对象。Gradle对象的数据类型就是Gradle。我们一般很少去定制这个默认的配置脚本。
- Project对象：每一个build.gradle会转换成一个Project对象。
- Settings对象：显然，每一个settings.gradle都会转换成一个Settings对象。

当我们执行gradle的时候，gradle首先是按顺序解析各个gradle文件。这里边就有所谓的顺序问题。这里看一下下面的文档，里面提到的生命周期就是说明这个顺序的。

**Lifecycle**

There is a one-to-one relationship between a Project and a build.gradle file. During build initialisation, Gradle assembles a Project object for each project which is to participate in the build, as follows:

- Create a [`Settings`](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html) instance for the build.
- Evaluate the settings.gradle script, if present, against the [`Settings`](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html) object to configure it.
- Use the configured [`Settings`](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html) object to create the hierarchy of Project instances.
- Finally, evaluate each Project by executing its build.gradle file, if present, against the project. The projects are evaluated in breadth-wise order, such that a project is evaluated before its child projects. This order can be overridden by calling `Project.evaluationDependsOnChildren()` or by adding an explicit evaluation dependency using `Project.evaluationDependsOn(java.lang.String).`

Project和`build.gradle`文件是一一对应的。在build初始化阶段，Gradle为每一个参与Build的project创建一个`Project`对象。

- 为此次build创建一个`Settings`实例
- Evaluate `settings.gradle`脚本，如果存在，修改前面`Settings`实例的属性
- 根据前面配置过的`Settings`实例创建`Project`实例的关系图
- 最后，通过执行它的`build.gradle`文件来evaluate每个Project。这个evaluate按照广度优先的顺序，这样可以让子Project在父Project之后evaluate。这个顺序也可以修改。

#### Project对象

每一个build.gradle文件都会转换成一个Project对象。在Gradle术语中，Project对象对应的是**Build Script**。

**Build scripts are code**

Project包含若干Tasks。另外，由于Project对应具体的工程，所以需要为Project加载所需要的插件(`apply plugin: 'xxx'`)。其实，一个Project包含多少Task往往是插件决定的。

所以，在Project中，我们要:

- 加载插件。
- 配置插件的行为，就相当于设置配置文件

Project API 文档: 

**https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html**

1. 加载插件

~~~ groovy
apply plugin: 'com.android.library'    <==如果是编译Library，则加载此插件
apply plugin: 'com.android.application'  <==如果是编译Android APP，则加载此插件
~~~

这是每个Android项目都会见到的代码，这个代码调用`apply`函数，看一下上面的文档，在`Method Summary`最底部才能看到`apply`函数，原来是继承自`PluginAware`接口。

![PluginAware apply方法](/img/gradle/apply.png)

那么，上面那个`apply`方法是上图所示的第几个方法呢？点击查看详细情况，发现第三个`apply`函数的解释是:

The following options are available:

- from: A script to apply. Accepts any path supported by Project.uri(Object).
- plugin: The id or implementation class of the plugin to apply.
- to: The target delegate object or objects. The default is this plugin aware object. Use this to configure objects other than this object.

除了最常见的`apply plugin: 'xxx'`,我们经常使用的`apply from: 'xxx'`也在此。

`apply plugin: 'xxx'`加载二进制插件，也就是把对应的jar包下载到了本地。至于我们使用的`apply from: 'xxx'`就是加载对应的文件。

2. 配置属性

这一点大家只要去看对应plugin的文档即可，文档内提到有哪些属性，则就可以配置哪些属性。

Gradle官方提供的plugin都可以在这里找到对应的文档（比如我们日常用的plugin: 'maven'）
**https://docs.gradle.org/current/userguide/userguide.html**

Android library和application文档见:
**http://google.github.io/android-gradle-dsl/current/index.html**

其他的第三方plugin,则直接找他们的文档即可。


Project 的一些其它说明:

##### Extra 属性

所有额外的属性都需要通过命名空间`ext`来定义。一旦额外属性被定义了，它可以被所属对象直接访问(读写)(在Project中定义的话，就可以在Project，Task和子Project中访问了)
只有初始声明的时候需要命名空间。(当然，如果和其它属性重名还需要全称来指定)

All extra properties must be defined through the `ext` namespace. Once an extra property has been defined, 
it is available directly on the owning object (in the below case the Project, Task, and sub-projects respectively) and 
can be read and updated. Only the initial declaration that needs to be done via the namespace.

最后就是查找顺序，放在这里供大家参考，就不翻译了。

##### Properties

A project has 5 property 'scopes', which it searches for properties. You can access these properties by name in your build file, or by calling the project's Project.property(java.lang.String) method. The scopes are:

- The `Project` object itself. This scope includes any property getters and setters declared by the Project implementation class. For example, Project.getRootProject() is accessible as the rootProject property. The properties of this scope are readable or writable depending on the presence of the corresponding getter or setter method.
- The `extra` properties of the project. Each project maintains a map of extra properties, which can contain any arbitrary name -> value pair. Once defined, the properties of this scope are readable and writable. See extra properties for more details.
- The `extensions` added to the project by the plugins. Each extension is available as a read-only property with the same name as the extension.
- The `convention` properties added to the project by the plugins. A plugin can add properties and methods to a project through the project's Convention object. The properties of this scope may be readable or writable, depending on the convention objects.
- The tasks of the project. A task is accessible by using its name as a property name. The properties of this scope are read-only. For example, a task called compile is accessible as the compile property.
- The extra properties and convention properties inherited from the project's parent, recursively up to the root project. The properties of this scope are read-only.
  When reading a property, the project searches the above scopes in order, and returns the value from the first scope it finds the property in. If not found, an exception is thrown. See Project.property(java.lang.String) for more details.

When writing a property, the project searches the above scopes in order, and sets the property in the first scope it finds the property in. If not found, an exception is thrown. See Project.setProperty(java.lang.String, java.lang.Object) for more details.

##### Dynamic Methods

A project has 5 method 'scopes', which it searches for methods:

- The `Project` object itself.
- The build file. The project searches for a matching method declared in the build file.
- The `extensions` added to the project by the plugins. Each extension is available as a method which takes a closure or Action as a parameter.
- The `convention` methods added to the project by the plugins. A plugin can add properties and method to a project through the project's Convention object.
- The tasks of the project. A method is added for each task, using the name of the task as the method name and taking a single closure or Action parameter. The method calls the Task.configure(groovy.lang.Closure) method for the associated task with the provided closure. For example, if the project has a task called compile, then a method is added with the following signature: void compile(Closure configureClosure).
- The methods of the parent project, recursively up to the root project.
- A property of the project whose value is a closure. The closure is treated as a method and called with the provided parameters. The property is located as described above.

参考: [深入理解Android之Gradle][1]

[1]: http://blog.csdn.net/innost/article/details/48228651