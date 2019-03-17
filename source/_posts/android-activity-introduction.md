---
title: Activity 简述
date: 2017-05-06 00:00:00
tags: 
	- android
---

`activity`是 `Android`系统四大应用组件之一，其它三个为 `Service`（服务） 、`BroadcastReceiver`（广播）、`ContentProvider`（内容提供器）。

1.  android中一个app的入口是`activity`，一个app至少要有一个`activity`，否则该app无法打开。

2.  `activity`通常是一个单独的窗口。

3.  `activity`一般通过`Intent`通信。

4.  所有的组件在使用时都要在Manifest中注册，activity注册方法为：

    ```xml
    <activity android:name=".MainActivity">
          <intent-filter>
                  <action android:name="android.intent.action.MAIN" />
                  <category android:name="android.intent.category.LAUNCHER" />
           </intent-filter>
    </activity>
    ```

    其中`intent-filter`为过滤器，其中声明表示了该`activity`响应主操作且置于`launcher`类别内，即打开app的时候会开启此`activity`。

    `<action>`元素指定这是应用的主入口点。`<category>`元素指定此`activity`应列入系统的应用启动器内，以便用户启动该activity。

## 生命周期

`activity`是向用户展示界面的类，同时所有的用户与手机的交互也大都在这个类里面进行初始化等一系列操作，同时`activity`也有不同的状态，可分为以下几类：

1.  `onCreate`
2.  `onStart`
3.  `onResume`
4.  `onPause`
5.  `onStop`
6.  `onDestroy`
7.  `onRestart`

以下是activity的生命周期流程图。

![activity生命周期](https://github.com/ruoyewu/ruoyewu.github.io/blob/master/assets/images/activity简介_1.png?raw=true)

### onCreate

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
```

`onCreate` 在 `activity` 第一次启动时调用，用于初始化 `activity` 。

这是基本的 `onCreate` 方法。可以看出，其中`setContentView(R.layout.activity_main)`方法将`activity`与界面布局联系到了一起，其中 `R.layout.activity_main` 是与`activity`相对应的布局文件，一般这个文件在 `res/layoout` 这个文件夹里面，可以说所有的布局文件，都存在于这个文件夹里面。同时还有一个参数``Bundle savedInstanceState``，这个参数用于`activity`意外退出之后的数据恢复。

### onStart

在这种状态下，`activity`处于非前台状态，但有可能是可见的，比如在一个`activity`中开启一个`dialogActivity`，那么`dialogActivity`处于前台状态，之前的`activity`可见，但是也仅限于可见，不能对其进行操作，称之为非前台状态。

### onResume

在这个阶段，`activity`真正处于与用户进行交互的阶段，每次`activity`从不可操作到可操作，都会调用这个方法，我们可以在这个函数里面进行一些需要自动刷新的事件。同时`activity`有时会频繁转入转出前台，这样为了界面转变的流畅性，不应在这里进行比较耗时的事件。

### onPause

与`onResume`相对应，当`activity`由前台转为非前台的时候，会调用这个方法。

### onStop

这时`activity`处于不可见状态，被视为处于“后台”，调用此方法之后，`activity`实例的所有变量信息将会保存，但无法执行任何代码。

### onDestory

`activity`结束，回收所有的内存，并在此阶段调用`onSaveInsanceState() `方法，用户结束时保存信息。

以下是各阶段的详细介绍。

![activity各阶段介绍](http://upload-images.jianshu.io/upload_images/2209096-e6733f865f65ee1a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 跳转

1.  显示跳转

    ```java
    Intent intent = new Intent(FromActivity.this,ToActivity.class);
    FromActivity.this.startActivity(intent);
    ```

    其中`FromActivity`是当前所在`activity`，`ToActivity`是要跳转到的`activity`。

2. 隐式跳转

    ```java
    Intent intent = new Intent();
    intent.setAction("com.intent.action.LOGIN");
    intent.addCategory("com.intent.category.LOGIN");
    startActivity(intent);
    ```

    其中的`action`和`category`要与在`Manifest`中声明的一致，用于从这些参数映射到相对应的`activity`。

    隐式跳转用于不知道某activity的名称，但知道其指定的action和category的activity，比如在app中打开相机拍照，我们一般使用的时候并不知道负责拍照的activity的名字，所以我们都是使用

    ```java
    int OPEN_CAMERA = 1;
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    intent.putExtra("output",uri);
    startActivityForResult(intent, OPEN_CAMERA);
    ```

    这段代码的意思就是你告诉系统，你需要调用一个相机，并把从相机中拍到的照片传到uri里面。一个手机里面可能有很多相机`activity`，比如手机自带的相机，或者一些其他什么的相机，这时系统就会弹出一个选择框，选出你要用的相机，同时也有可能某位同学手抖了抖，把手机root之后删掉了系统自带的相机，这时系统就会向你抱怨说，它没有相应的程序来用。

    由此我们可以看出，使用隐式跳转可以更方便的对手机功能进行扩展，我们只需要写一份筛选性质的代码，让系统把所有符合我们条件的`activity`列出来，我们再在其中选择。