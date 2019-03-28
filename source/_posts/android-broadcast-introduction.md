---
title: Android 广播介绍
date: 2019-03-26 13:30
tags:
	- android
---

广播是 Android 的四大组件之一，是观察者模式的一种实现，利用广播能够轻松的进行应用内不同组件的通信、应用之间的通信等，一个广播分为两个部分，广播发送者和广播接收者，一般来说充当发送者的是 Context ，在 Context 中有`sendBroadcast()`、`sendOrderedBroadcast()`等方法，只需要调用这些方法就能开始一个广播的发送，而广播接收者是需要开发者自己注册的，广播接收者的注册有两种方式，静态注册和动态注册。静态注册即在`AndroidManifest.xml`文件中增加一个 receiver 标签，例如下面：

```xml
<receiver android:name=".MyReceiver">
    <intent-filter>
        <action android:name="android.net.conn.CONNECTIVITY_CHANGE"/>
    </intent-filter>
</receiver>
```

或者是在代码中动态注册一个接收者：

```java
IntentFilter filter = new IntentFilter();
filter.addAction("android.net.conn.CONNECTIVITY_CHANGE");
MyReceiver receiver = new MyReceiver();
registerReceiver(receiver, filter);
```

两种注册方式导致的接收者的生命周期是不同的，使用静态方式注册的接收者，即便是应用没有启动接收者也可以工作。而使用动态注册的接收者，需要应用启动并执行到对应的代码，完成注册之后接收者才能工作，并且动态注册的广播一般还需要手动注销接收者，因此动态方式下，可以更加灵活的使用广播接收者。

广播按照定义者可以分为两种，系统广播和应用广播，系统广播由系统内部发送，比如上面的这个广播就是一个系统广播，会在网络状态更换的时候发送这样一个广播，其他的还有比如截屏等。这种广播一般只需要定义广播接收者即可，发送广播是系统内部的事。应用广播则是开发者在应用中定义的广播类型，当用户在应用中执行了某个操作后，可以通过发送广播的方式将这个事件传递给所有的广播接收者。在其他应用中也可以定义一个接收应用广播的接收者，可以用于接收其他应用发送的广播，所以这也能够完成一定程度的进程通信。

为了更够完成进行通信，所以在发送广播的过程中也使用到了 Binder 机制，当在 Context 调用了一个发送广播的方法之后，就会通过 Binder 通信调用 ActivityManagerService 中相应的方法，这时就进入了系统进程工作，在这个进程中负责处理所有的广播相关事宜，收到一个发送广播的请求之后，AMS 就会在内部寻找所有的接收者，并再通过 ActivityThread 进入每个用户进程，将广播传递给每个进程的接收者，再 AT 内部又会首先利用 Handler 机制切换到主线程，最后调用 Receiver 的`onReceive()`方法完成一次广播的发送。

广播本身也分为几类，普通广播、有序广播、本地广播和粘性广播。

### 普通广播

普通广播就是直接使用`sendBroadcast()`方法发送的广播，默认广播是异步接收的，当广播发送出去之后，各广播接收者会同时接收到这个广播。使用广播之前首先要定义一个广播接收者，继承于 BroadcastReceiver 。BroadcastReceiver 是一个抽象类，拥有一个抽象方法`onReceiver()`，自定义广播接收者的时候，只需要实现这个方法就行了。

```java
public class MyReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {

    }
}
```

`onReceive()`方法有两个参数，Context 和 Intent ，其中 Intent 参数可以用于发送广播的时候传参，然后在这个方法中根据 Intent 传入的不同参数执行不同的操作。由于默认情况下广播接收者也是直接运行于主线程的，所以也不能在这个方法里做耗时操作，否则会引起 ANR 。

### 有序广播

有序广播的使用方式与普通广播一致，但是多了“有序”这个概念，即当一个有序广播发送出去的时候，这个广播并不是直接发送给所有的接收者，而是首先发送给优先级最高的广播，然后再发送给优先级次高的广播，不过如果仅仅拥有这个功能，那它与普通广播也没有太大的区别，更重要的特点是优先级高的接收者收到一个广播之后，可以决定是否将这个广播继续传递给优先级，一个接收者可以通过`abortBroadcast()`方法停止一个广播的传递，优先级低的接收者就无法再接收到这个广播了。另外，还可以利用`setResult()`和`getResultXXX()`方法完成有序广播之间的数据传递，优先级高的接收者可以将自己的执行结果传递给下一优先级的接收者，完成一种链式的操作。

首先，发送有序广播需要使用`sendOrderedBroadcast()`方法，其次，要给每一个接收者设置优先级，优先级的设置也分为静态和动态：

```xml
<receiver android:name=".MyReceiver">
    <intent-filter priority="100">
        <action android:name="android.net.conn.CONNECTIVITY_CHANGE"/>
    </intent-filter>
</receiver>
```

```java
IntentFilter filter = new IntentFilter();
filter.addAction("android.net.conn.CONNECTIVITY_CHANGE");
filter.setPriority(100);
MyReceiver receiver = new MyReceiver();
registerReceiver(receiver, filter);
```

### 本地广播

本地广播顾名思义，就是这个广播只会在本应用之中传递，这样就能够避免应用之间的广播污染，只在本应用内传递广播，就不需要进程间通信的功能，所以本地广播并不会通过 ActivityManagerService 完成，而是通过 LocalBroadcastManager 完成。这个类提供了诸如`registerReciever()`和`sendBroadcast()`等注册接收者和发送广播的方法，并且，本地广播只能使用 LBM 注册，并不支持静态注册，简单的说，本地广播就是一个观察者模式的本地实现，在 LBM 中保存了所有注册的接收者，当发送广播的时候，就会根据 IntentFilter 选择合适的接收者，调用其`onReceive()`方法。

### 粘性广播

粘性广播的基本使用与普通广播一致，不过粘性广播多了一个特性，就是使用`sendStickyBroadcast()`发送的粘性广播，在提交给所有的接收者之后并不会消失，在系统中始终保存着最后一个发送的粘性广播，并且在有一个新的广播接收者注册完成之后会收到这个广播，也就是说虽然注册发生在发送广播之后，但是接收者仍然能够接收到最后一个发送的粘性广播。

不过在最近的版本中（我也不知道从哪个版本开始）这个广播被弃置了，原因是它的安全性得不到保证，任何人都可以访问，任何人都可以修改。

### 总结

广播是 Android 中的四大组件，其基本秉承的是观察者模式的思想，通过 ActivityManagerService（不愧是 Android 中的大管家）完成广播的中转：除了本地广播，所有的接收者都要到这里注册，然后发送广播也是先发送到这里，再由 AMS 分发出去。广播的有点，也就是观察者模式的优点，就是让发送者和接收者解耦，它们之间的所有的联系都是使用中介者完成的，经由中介者统筹，发送者与接收者双方可以更加专注于自己的任务，还有一个优点就是在通信上，不需要知道接收者的情况，发送者仅需要一个 Context 实例，就可以在任何地方发送一个广播，最终都能被接收者收到。