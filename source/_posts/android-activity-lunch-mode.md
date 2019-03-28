---
title: Android Activity 启动模式
date: 2019-03-28 9:00
tags:
	- android
---

Activity 有四种启动模式，分别用于不同的使用场景：

1.  standard
2.  singleTop
3.  singleTask
4.  singleInstance

下面将介绍每种启动模式的特点、使用场景以及使用方式。

### Standard

这是 Android 中默认的 Activity 启动模式，每次启动一个 Activity 的时候，都会在当前的创建一个新的 Activity 实例加入到当前的活动栈栈顶。这种模式下没有 Activity 的复用功能，启动了多少次 Activity 就会创建多少个 Activity 实例，一般情况下都会使用这种方式。

### SingleTop

这种模式下，如果要启动一个 Activity ，但恰巧这个 Activity 的实例正好在当前活动栈的栈顶（就是当前的 Activity），此时就满足 singleTop 的复用条件，系统不会再新建一个 Activity 实例，而是调用这个活动的`onNewIntent()`方法，标志复用了这个 Activity，否则，就会创建一个实例加入栈顶 。

这种启动模式，一般用于消息列表页面，登录页面等，这些页面一般都可以从外部多次打开，所以可以采用 Activity 复用，因为它们展示的东西、拥有的功能并没有什么不同，只不过是由于外部的重复启动。

一般情况下有两种方式设置一个 Activity 的启动模式，一种是直接在`AndroidManifest.xml`文件中 activity 标签中声明这个活动的启动模式，如下：

```xml
<activity android:name=".ui.home.MainActivity"
    android:launchMode="singleTop">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
        <action android:name="android.intent.action.VIEW"/>
    </intent-filter>
</activity>
```

另一种则是在代码中启动一个 Activity 的时候，给 Intent 添加 FLAG 选择启动这个 Activity 的启动模式，如：

```java
Intent intent = new Intent();
intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
```

### SingleTask

使用这种模式，如果要启动的 Activity 在当前的活动栈内，那么就会不断对当前的活动栈执行出栈操作，直到这个 Activity 成为栈顶，才会停止，然后调用它的`onNewIntent()`方法表示复用。在这种模式下的 Activity 复用异常霸道，如果自己不在栈顶，就会将所有在自己之上的 Activity 都挤出栈。

这种启动模式，可以用来完成连续退出多个连续的 Activity 的情况，比如从主页面开始打开了好几个 Activity ，那么如果能够一步回到主页面？如果不使用启动模式，就需要在应用中有一个活动管理器，可以负责多个 Activity 的退出，但是也可以很方便的使用 SingleTask 启动模式，使用这种模式打开主页面，系统就会自动完成这些 Activity 的出栈。或者这种方式也可以用来避免页面之间循环启动而浪费内存的情况，如有活动 A B ，活动 A 可以启动 B ，B 又可以启动 A ，那么这么连续启动几次之后就会有大量的 Activity 实例，如果这些实例的功能并没有不同的话，就可以使用 SingleTask 模式完成活动的复用，例如微信的聊天页面，点击用户头像进入用户主页，然后又可以点击开始聊天进入聊天界面，但是两个聊天界面的功能完全一样，此时就可以使用 SingleTask 模式，使得在已有聊天界面的情况下复用这个 Activity 。

对应的配置方法如下：

```xml
<activity android:name=".ui.home.MainActivity"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
        <action android:name="android.intent.action.VIEW"/>
    </intent-filter>
</activity>
```

```java
Intent intent = new Intent();
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP|Intent.FLAG_ACTIVITY_SINGLE_TOP);
```

使用代码设置的时候需要同时使用两个标志位，如果仅使用第一个 Intent.FLAG_ACTIVITY_CLEAR_TOP 的话，这个 Activity 本身也会被销毁并重新创建，加上 Intent.FLAG_ACTIVITY_SINGLE_TOP 之后，不会重新创建而是调用其`onNewIntent()`方法。

### SingleInstance

该模式下的 Activity 只会单独的存在于一个活动栈中，当在一个 Activity 中要启动这种模式的 Activity 的时候，首先会检测所有的活动栈，判断是否有这样一个 Activity ，如果有，则将这个活动栈移到活跃状态，并调用活动的`onNewIntent()`，如果没有，就会重新创建一个活动栈，并在其中创建一个活动实例，另外，这个模式下的 Activity 启动一个其他的 Activity 的时候，不会将活动添加在这个栈上，而是放在上一个非 SingleInstance 活动存在的活动栈中。

也就是说，使用 SingleInstance 模式启动的 Activity ，在整个系统中是以单例的形式存在的，这种启动模式一般用于开启一个不打乱原先的活动栈的页面，如呼叫来电页面，这是一种突发性事件，也与原来的页面没有关联，类似于代码中“工具类”的存在，这就可以使用这个单例 Activity 。

配置如下：

```xml
<activity android:name=".ui.home.MainActivity"
    android:launchMode="singleInstance">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
        <action android:name="android.intent.action.VIEW"/>
    </intent-filter>
</activity>
```

```java
Intent intent = new Intent();
intent.setFlags(Intent.ACTIVITY_NEW_TASK);
// 这种方式会新建一个活动栈用于启动 Activity ，但并不能保证这个活动启动的其他活动不会存在于这个活动栈上
```

以上是常见的的四种启动模式的相关介绍，但是 Activity 的启动模式远不止如此，看 Intent 类内部的源码就可以看到，所有 FLAG_ACTIVITY_ 开头的其实都是与启动模式相关的标志，并且每种标志叠加在一起，代码设置的 FLAG 与 xml 中设置的 lanuchMode 叠加在一起，会产生不同的效果，具体的还要具体分析，本文只是就四种 lanuchMode 进行介绍。