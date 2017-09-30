---
title: Android管理任务栈之——launchMode
date: 2016-03-06
tags:
---
## 前言

之前遇到了一个问题，大概是有4个Activity，分别为A, B, C, D Activity，当你的Activity跳转, A -> B -> C -> D 跳转，此时，你需要从D跳回B，不能把C`finish()`掉，因为用户中途可能会返回C，而不继续进行下一步操作；直接跳也不可以，因为默认情况下跳到B的时候，用户点返回会返回D而不是A，这个时候我知道了LaunchMode，发现这种情况很适合一个LaunchMode的使用 ———— `singleTask`，趁此机会，研究一下LaunchMode。

## LaunchMode 开篇

说到LaunchMode，我们来看看官方文档怎么说 ：

> An instruction on how the activity should be launched. There are four modes that work in conjunction with activity flags (`FLAG_ACTIVITY_*` constants) in Intent objects to determine what should happen when the activity is called upon to handle an intent. They are:

- "standard" 

- "singleTop" 

- "singleTask" 

- "singleInstance"

  The default mode is "standard".

即Activity有4种启动模式，分别是

- "standard" 
- "singleTop" 
- "singleTask" 
- "singleInstance"

这个需要在Manifest中指定。可以和Intent中的`FLAG_ACTIVITY_* `结合使用。

当然，我们要明确一点，Android中以栈的形式来管理`Activity`，至于什么是栈，请复习数据结构相关内容，这里假设大家都理解栈。

## 四种模式各个模式的作用

先告诉大家一个方法，通过如下命令获取你当前包的`Activity`任务栈

~~~ shell
adb shell dumpsys activity activities | sed -En -e '/Stack #/p' -e '/Running activities/,/Run #0/p' | grep "包名" 
~~~

- "standard"

大家默认新建的`Activity`就是这个模式的，这个模式相当于不对`Activity`加任何约束，即你调用`intent.startActivity()`的时候，每调用一次，就会在栈中新加一个改`Activity`。



先看一下我们当前的任务栈：

~~~ shell
Run #7: ActivityRecord{ac97db9 u0 com.air.activitylaunchmodetestdemo/.MainActivity t1211}
~~~

可以看到，当前只有一个`MainActivity`，此时，当我点击按钮执行`startActivity(new Intent(this, MainActivity.class))`的时候，可以看到任务栈变成了：

~~~ shell
Run #8: ActivityRecord{11e0192c u0 com.air.activitylaunchmodetestdemo/.MainActivity t1211}
Run #7: ActivityRecord{ac97db9 u0 com.air.activitylaunchmodetestdemo/.MainActivity t1211}
~~~

因为没有在`startActivity()`之后执行`finish()`方法，现在我们有了两个`MainActivity`实例，此时如果退出应用，要点两次后退键才可以。如果我再次点击按钮执行`startActivity(new Intent(this, MainActivity.class))`的时候，可以想象新的任务栈该变成什么了。

 接下来我们以图的形式说明该LaunchMode情况下，任务栈的样子。

 ![ActivityStack--standard](/img/launchMode/ActivityStack--standard.png)

就像上图画的那样，每启动一个新的Activity，就会添加到栈上，无论即将启动的Activity是否是同一个Activity。



- "singleTop"

看到这个名字，先靠命名理解一下，在顶部的时候是单一存在，因为是以栈的形式来管理`Activity`的，那么顶部指的就是栈的顶部。

好了，上面的猜测其实已经和真正的情况是差不多了，我在此再补充几点细节。对于不是栈顶的`Activity`，即使设置了这个launchMode，依旧会创建新的Activity。例如，存在两个`Activity`A、B，它们的launchMode都是singleTop，如果你在A中启动A，栈不会新添加`Activity`，因为此时A`Activity`在栈顶，但是，如果你用A启动B，B就会被加入到栈中，此时栈就变成了AB，再拿B启动A，会产生A的新实例添加到栈内，现在栈变成了ABA。

拿前面举的A`Activity`例子来说，在A中启动A，由于A在栈顶，所以栈内不会添加A的新实例，自然不会调用A的`onCreate()`方法，这个时候，如果启动A的时候给A传了额外的参数，那么需要在哪里处理呢？答案是`onNewIntent()`，这里就是singleTop和standard两种launchMode不同的地方。



常用场景：带搜索功能的`Activity`，搜索不同的关键词不用多次启动同一个`Activity`。



- "singleTask"

上面介绍的两个launchMode的作用不能实现栈中只存在一个`Activity`实例，但是有些时候，就像我在开头举的那个例子，业务逻辑中间要跳好几层，最终只想跳到入口界面，即栈中只需要存在一个相应的`Activity`，而不是多个。



总结一个singleTask的作用，若一个`Activity`的launchMode被指定为singleTask，则在对应的栈中只会存在一个实例。若这个实例不存在，则新建实例；若存在，则把在栈中该实例之上的其他实例移除出栈，并调用该`Activity`的`onNewIntent()`方法，将intent传入。 

![ActivityStack--singleTop](/img/launchMode/ActivityStack--singleTop.png)



我们来通过代码验证一下：启动了四个`Activity`，

~~~ shell
Run #7: ActivityRecord{b53e419 u0 com.air.activitylaunchmodetestdemo/.BActivity t3909}
Run #6: ActivityRecord{7a83deb u0 com.air.activitylaunchmodetestdemo/.AActivity t3909}
Run #5: ActivityRecord{df9c4ad u0 com.air.activitylaunchmodetestdemo/.CActivity t3909}
Run #4: ActivityRecord{e7c438 u0 com.air.activitylaunchmodetestdemo/.MainActivity t3909}
~~~

CActivity为launchMode为singleTask的`Activity`，此时，从BActivity启动CActivity，通过前面的介绍，我们猜测会调用CActivity的`onNewIntent()`方法，并且，在栈中，CActivity之上的`Activity`都会被移除，此时，执行`startActivity()`方法，最新的栈内容如下：

~~~ shell
Run #5: ActivityRecord{df9c4ad u0 com.air.activitylaunchmodetestdemo/.CActivity t3909}
Run #4: ActivityRecord{e7c438 u0 com.air.activitylaunchmodetestdemo/.MainActivity t3909}
~~~

可以看出，确实是我们所介绍的效果。至于设置taskAffinity后该launchMode的表现，我们在这篇文章中不会涉及，留到下一篇文章再与大家见面。





常用场景：注册Activity或者启动应用的首个`Activity`，这样就不用担心任务栈内有其他的`Activity`没有销毁了。



- "singleInstance"

这个或许可以称之为singleTask模式的加强版，对于设置了singleTask的`Activity`，我们可以让它加入到在我们当前的任务栈中，也可以新建一个任务栈，把它放入(设置taskAffinity的情况下)。但是，对于singleInstance的`Activity`，我们只能新建一个任务栈，并把它放入，没有任何`Activity`可以和它在一个栈中。



 那么对于我们开发的应用，其中有一个`Activity`需要给其他应用使用，比如浏览器，你从自己的应用点开一个链接，使用浏览器应用开启网页，你单机后退，直接退到了你的应用，而不是你之前浏览的其他页面，如果你在浏览器中访问的连接可以访问你的应用，那么，此时点击后退你会发现，居然在你应用所在的栈中进行了回退操作，而不是退到浏览器页面，然后你退出应用，发现这个时候反倒打开了浏览器页面，这样说有点绕，我们以图的形势来展示一下。

![ActivityStack--singleInstance](/img/launchMode/ActivityStack--singleInstance.png)

至于常用场景，我本以为支付、分享之类的很可能会使用，但是我反编译微信查看其manifest文件的时候，发现它并没有使用这个launchMode，所以暂时这里空着，如果你有什么使用场景，可以给我留言。



参考文章：

1. [官方文档][1]
2. [深入讲解 Android 中的 Activity launchMode][2]

参考书籍：

1. 《第一行代码》
2. 《Android开发艺术探索》



[1]: http://developer.android.com/guide/topics/manifest/activity-element.html#lmode
[2]: http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/