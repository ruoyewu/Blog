---
title: Android Service 简述
date: 2019-03-09 11:40
tags:
	- android
---

Service 是 Android 中的四大组件之一，相比于多运行在前台用户可见的 Activity ，Service 的存在用户往往察觉不到，如果说 Activity 主要是负责与用户进行交互的话，那 Service 则更多的用来处理任务，如一些后台文件的下载，或者是大量数据的处理等，用户提交给 Service 一个任务之后就可以继续与 Activity 交互，顺便等待 Service 完成任务。

### 启动方式

Service 有两种启动方式，start 模式和 bind 模式。

#### 生命周期

两种启动方式对应的 Service 的生命周期是不同的，在 start 模式中，生命周期为：

`onCreate() -> onStartCommand() -> onDestroy()`

而在 bind 模式中，生命周期为：

`onCreate() -> onBind() -> onUnbind() -> onDestroy()`

#### 功能

在 start 模式中，启动一个 Service 就是调用`startService(Intent)`，此时会开启一个 Service ，并调用 Service 的`onCreate()`方法，并随即调用`onStartCommand(Intent, int, int)`方法，一般就会在这个方法里根据 Intent 携带的一些值来执行相关的工作，在这种方式下，Activity 与 Service 之间的交流只能通过`startService()`方法完成，而且这是一个单向的，一旦 Activity 开启一个 Service 并将任务交给 Service 之后，Activity 便不会再得知 Service 的任务执行情况，也不会知道什么时候 Service 的任务会执行完成，什么时候 Service 会销毁。与这种启动方式对应的关闭 Service 的方法是`stopService(Intent)`，或者也可以由 Service 自行确定什么时候应该关闭自己：可以在 Service 执行完某个任务之后自身调用`stopSelf()`方法完成 Service 的关闭。

在 bind 模式中，启动一个 Service 需要调用`bindService(Intent, ServiceConnection, int)`开启 Service ，相比于使用 start 方式启动 Service 来说，多了两个参数，一个是 int 类型的 flag ，对 Service 的一些配置做了规定，如 BIND_AUTO_CREATE 、BIND_DEBUG_UNBIND 等，另一个则是 ServiceConnection 接口：

```java
public interface ServiceConnection {
    void onServiceConnected(ComponentName name, IBinder service);
    void onServiceDisconnected(ComponentName name);
    default void onBindingDied(ComponentName name) {
        
    }
    default void onNullBinding(ComponentName name) {
        
    }
}
```

接口中有四个方法，根据名字就可以看到，这是与 Service 的连接的回调。也就是说，使用 bind 启动 Service 的时候是需要添加一个回调接口的，这个接口可以将 Service 的状态告知给 Activity ，这也就意味着，使用 bind 开启一个 Service 之后，Activity 不再是对 Service 的状态一无所知了。另外，在 ServiceConnection 的`onServiceConnected()`有两个参数，其中一个参数是 IBinder 类型的，Service 有一个抽象方法`onBind()`：

```java
@Nullable
public abstract IBinder onBind(Intent intent);
```

在这里返回了一个 IBinder 实例，实际上这里返回的 IBinder 实例就是`onServiceConnected()`方法的参数，这个 Binder 可以完成 Activity 到 Service 的通信，一般的使用方法是：在 Service 中定义一个 Binder 的子类，然后在`onBind()`方法中返回这个子类的实例，我们可以在这个子类中定义一些控制 Service 中任务执行的方法，如开始下赞、暂定下载等。例子如下：

```java
class DownloadService extends Service {
    // 省略其他方法
    
    @Override
    public IBinder onBind(Intent intent) {
        return new DownloadBinder();
    }
    
    class DownloadBinder extends Binder {
        public void startDownload() {
            
        }
        
        public void pauseDownload() {
            
        }
    }
}
```

如此依赖，使用 bind 模式时启动的 Service 就能与 Activity 进行一定程度上的数据交流了。此时关闭 Service 的方式是调用`unBindService(ServiceConnection)`方法。

对于一个 Service 来说，它可以同时使用 start 模式和 bind 模式启动，此时 Service 所具有的功能仍旧与上述相同，不过 Service 的周期略有不同，当一个 Service 尚未创建时，无论是调用`startService()`还是调用`bindService()`都会先创建一个 Service ，调用`onCreate()`方法，之后的话再根据调用方式不同分别调用`onStartCommand()`和`onBind()`方法，对于一个使用两种方式开启的 Service ，只有`stopService()`和`unBindService()`这两个方法都调用了之后才会销毁 Service 。

### Service 类别

Service 一般有两种：后台服务和前台服务。两种服务在优先级上有所不同，在表现形式上也有所不同。

#### 优先级

Android 中有对 Activity 和 Service 分优先级，当系统内存不足时，会首先回收优先级低的服务，所以有时为了能让服务保持一直运行，就可以将服务设置成前台服务。

#### 表现

Service 一般用于后台执行某项任务，所以在用户看来不像是 Activity 那样有一个确切的界面。不过有时可能需要某些服务常驻并且显示服务执行的某些结果，如各种天气软件，有的会在通知栏中常驻一个通知，时刻显示当前的天气状况，这就可以使用前台服务实现。前台服务对外的表现类似于通知，会在状态栏有一个通知标记，在通知栏展示更多的详细信息，然后再服务运行的过程中，可以不断刷新数据并将结果通过通知栏展示出来。

使用前台 Service 时需要调用 Service 的`startForeground(int, Notification)`方法，这里提供了一个 Notification 作为参数，可以设置通知栏中展示的界面，开启服务之后就会有一个通知常驻通知栏，服务也就变成了前台服务。

### 跨进程 Service

Service 也可以用来进行跨进程通信，如一个 APP 的某个 Activity 通过绑定另一个进程的 Service 完成两个进程之间的数据交流。使用 Service 跨进程的时候需要使用到 AIDL ，AIDL 是 Android 中一种跨进程的方式，更具体的此后再说。

### Service 与 Thread

在功能上，后台服务可能会与子线程有点像，都是用于后台执行一些耗时任务。那么二者区别是什么？

首先，Service 默认情况下是运行在主线程的，所以要使用 Service 执行耗时操作一样离不开开启子线程，特别的还有一种专门集成了子线程的类 IntentService ，一般问出上面这种问题的人，可能真正想知道的是在 Activity 中开启子线程和在 Service 中开启子线程的区别。

在执行能力上可以确定两种方式创建的子线程没有什么区别，区别就在于生命周期上，如果在 Service 中创建子线程，因为所有的 Activity 都可以同这一个 Service 绑定，进而操控子线程，且 Service 的生命周期可以贯穿整个应用运行过程，相比之下，如果在 Activity 中创建子线程，一方面只有当前 Activity 可以操控这个线程，另一方面一个 Activity 的生存时间有限，随着 Activity 的销毁，一方面要担心是否会出现内存泄漏，一方面也无法在操控这个子线程。所以在这种运行时间比某一个 Activity 生命周期长的子线程的使用中，可以选择在 Service 中创建子线程，而每一个 Activity 都可以通过与 Service 绑定的方式参与到子线程的控制中。

### 总结

就本地 Service 而言，可以分为后台服务和前台服务，Service 默认是执行在主线程的，所以如果有耗时操作要执行还是需要创建子线程，这种方式较之于直接在 Activity 中创建子线程，在某些情况下是有优势的。另外，Service 还有一个子类 IntentService ，这个类内部集成了创建子线程的工作，并且集成的是运行着 Handler 机制的 HandlerThread ，在许多耗时操作中直接就可以使用 IntentService 。