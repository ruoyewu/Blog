---
title: Android Service 启动流程
date: 2018-05-09 21:40
tags:
	- android
---

在 Android 中启动 Service 有两种方案，一种是使用 Context.startService 方法，一种是通过 Context.bindService 方法，两种方法启动 Service 的生命周期不同，所能提供的交互程度也不同。

- startService
  - 生命周期：onCreate → onStartCommand → onDestroy
  - 交互：通过 onStartCommand 方法向 Service 传递信息
- bindService
  - 生命周期：onCreate → onStartCommand → onBind → onUnbind → onDestroy
  - 交互：通过 IBinder 接口进行进程间通信，通过 onStartCommand 方法传递信息

两种方法启动 Service 的具体过程也有些区别：

## StartService 流程

> 缩写：
>
> ActivityManagerService：AMS
>
> ActiveServices：AS
>
> ActivityThread：AT
>
> ApplicationThread：APT

### 方法调用过程

#### ContextImpl.startService

设置所要开启的服务的基本配置和信息，类型、是否前台等。通过 Binder 调用 AMS.startService 方法，进入 system 进程。

#### AMS.startService

判断 Intent 和调用方包名可用性，获得当前进程 uid 和 pid ，调用 AS.startServiceLocked ，参数增加。

#### AS.startServiceLocked

判断调用方为前台或后台，调用 AS.retrieveServiceLocked 得到 ServiceLookupResult ，进而得到 ServiceRecord （没有时会创建）。

验证用户，验证权限（会开启权限验证 Activity ），延迟启动处理。调用 AS.startServiceInnerLocked 方法。

#### AS.startServiceInnerLocked

电量记录，调用 AS.bringUpServiceLocked 方法，处理结果。

#### AS.bringUpServiceLocked

判断 Service 是否已经开启，判断是否在等待重启，延迟判断，用户判断，进程判断（如果目标进程没有开启，先去开启进程，调用 AMS.startProcessLocked 方法），调用 AS.realStartServiceLock ed 方法。

#### AS.realStartServiceLocked

调用 AS.bumpServiceExecutingLocked 方法完成埋雷操作，调用 APT.scheduleCreateService 进入目标进程的 binder 线程，走向 Service.onCreate 方法。

#### APT.scheduleCreateService

调用 AT.sendMessage 进入目标进程主线程，并通过 CREATE_SERVICE 字段进入 AT.handleCreateService 方法。

#### AT.handleCreateService

为 Service 分配 Context ，调用 Service.onCreate 方法，执行用户在此方法里写的代码，完成初始化。将 Service 加入 Map 。再调用 AMS.serviceDoneExecuting 方法进入 system 进程。

#### AMS.serviceDoneExecuting

进入 system 进程，检测

#### AS.realStartServiceLocked

调用 AS.sendServiceArgsLocked 走向 Service.onStartCommand 方法。

#### AS.sendServiceArgsLocked

获取 Service 启动参数，同样调用 AT.scheduleServiceArgs 方法进入目标进程，然后通过 SERVICE_ARGS 字段进入 AT.handleServiceArgs 方法进入目标进程主线程调用 Service.onStartCommand 方法，执行用户代码。

### 总体流程

#### 进程角度

1. 首先从用户进程发起开启服务请求，通过 Binder 进入到系统进程
2. 系统进程里负责为服务创建进程（如果服务是存在于其他进程），然后构建服务运行所需的一些环境，为 ANR 埋雷等，然后再通过 Binder 进入到服务所在进程
3. 进入服务所在进程，通过 Handler 机制进入到主线程（服务运行在主线程），然后在这个线程里调用 Service.onCreate Service.onStartCommand 等方法（一般用户会重写这些方法用来执行自定义的工作），最后再通过 Binder 回到系统进程
4. 完成服务的开启工作后在这个进程里做一些后续工作，如为 ANR 拆雷等。

#### 类角度

1. ContextImpl： 执行 startService stopService 等方法的入口
2. ActivityManagerService：运行在系统进程，是一个 Binder 类，作为其他进程执行功能的入口
3. ActiveServices：服务管理的相关类，实际执行服务的创建、销毁等工作，一般被 AMS 调用相关功能
4. ApplicationThread：运行在目标进程，是一个 Binder 类，用于接收系统进程消息，然后通过 Handler 交由主线程执行工作
5. ActivityThread：进程的主线程所在地，启动主线程 Looper ，并通过 Handler 接收消息然后相应执行，在这里面调用 Service.onCreate 等方法
6. ServiceRecord：表示一个运行的应用服务，继承自 Binder ，存放服务相关信息，在进程切换的过程中随之转移。

如图：

![](http://blog-1251826226.coscd.myqcloud.com/start_service_process_class.jpg)

## BindService 流程

> 缩写：
>
> ActivityManagerService：AMS
>
> ActiveServices：AS
>
> ActivityThread：AT
>
> ApplicationThread：APT
>
> IServiceConnection：ISC

### 方法调用过程

#### Context.bindService

包装更多参数，调用 Context.bindServiceCommon 方法。

#### Context.bindServiceCommon

在这里调用 LA.getServiceDispatcher 方法将 ServiceConnection 封装成 ISC ，使之用于后续的进程间通信，然后调用 AMS.bindService 方法，进入系统进程。

#### AMS.bindService

对 Intent 和 调用方包名判断，并调用 AS.bindServiceLocked 方法。

#### AS.bindServiceLocked

AS.startServiceLocked 方法功能类似，对调用方进程判断，对调用方 Activity 进行判断（Service 只能与 Activity 绑定），判断权限（可能会开启权限验证窗口），将回调类 IServiceConnection 封装成 ConnectionRecord 保存，然后调用 AS.bringUpServiceLocked 方法启动 Service 。

#### AS.bringUpServiceLocked

与 startService 流程一样，最终都会走到这个方法开启一个 Service ，不同的是两种方式开启 Service 提供的参数不一样，导致开启之后的工作略有差异。

在方法中进行了 Service 是否已经开启、进程是否开启等判断，最终进入 AS.realStartServiceLocked 方法。

#### AS.realStartServiceLocked

埋雷，调用 APT.scheduleCreateService 方法走向 Service.onCreate 方法。接下来的操作对于 startService 和 bindService 两种开启方式来说略有不同，下面两个方法会走向两个 Service 的生命周期：

- AS.requestServiceBindingLocked：以 bindService 方法启动
- AS.sendServiceArgsLocked：以 startService 方法启动

对于使用 bindService 方法启动方式来说，它会调用 AS.requestServiceBindingLocked 方法。

#### AS.requestServiceBindingLocked

这里面也会先调用 AS.bumpServiceExecutingLocked 埋雷，然后调用 APT.scheduleBindService 进入目标进程调用准备调用 Service.onBind 方法。

#### AT.handleBindService

调用 Service.onBind 方法得到 IBinder 对象（之后通过系统进程作为中转，将其传递给调用进程），这个 IBinder 类是由开发者写的，并在自定义的 Service.onBind 方法中构造实例，用于其他进程使用服务时的媒介，一般会在 Binder 类中定义 Service 提供的功能等，使用者也会定义一个同样的 Binder 类，然后调用里面的方法就可以通过这个对象获取 Service 提供的服务。

得到 IBinder 实例之后，AT 还要将这个实例交给系统进程，使用 AMS.publishService 方法。这里还有一个判断，如果用户进程多次调用 Context.bindService ，后续的只会调用 Service.onRebind 方法。

#### AMS.publishService

调用 AS.publishServiceLocked 方法。

#### AS.publishServiceLocked

找到 Service 对应的 ConnectionRecord ，并调用 ISC.connected 方法，这里的 ISC 就是用户调用 Context.bindService 时传递的 ServiceConnection 封装。并调用了 AS.serviceDoneExecutingLocked 方法完成拆雷。

#### ISC.contected

ISC 实现类是 LoadeApk.ServiceDispatcher.InnerConnection ，调用了 LoadeApk.ServiceDispatcher.connected 方法。

#### LoadeApk.ServiceDispatcher.connected

使用 Handler 跨线程到主线程或者直接在本线程调用 LoadeApk.ServiceDiapatcher.doConnected 方法。

#### LoadeApk.ServiceDiapatcher.doConnected

用户的 ServiceConnection 对象就是保存在这个类里面的，在这个方法里，调用了 ServiceConnection 的各个生命周期方法。如调用 ServiceConnection.onServiceConnected 方法，将 Service 提供的 Binder 对象一步步从 Service 所在进程传递到了用户所在进程，用户得到这个 Binder 之后，之后与 Service 的部分通信就可以直接通过这个 Binder 跨进程通信，而不需要系统进程再作中转，不过像 unbindService 这种操作还是要通过系统进程完成 Service 的销毁的。

总之，到这个方法，回调了用户进程的方法，完成了 Service 的绑定和 Binder 的传递，绑定过程结束。

### 总体流程

#### 进程角度

1. 从用户进程发起绑定请求，通过 AMS 进入系统进程
2. 在系统进程中完成创建 Service 所做的准备，并在创建结束之后进入目标进程
3. 目标进程中调用 Service 的各个生命周期方法，并得到 Service.onBind 返回的 Binder ，然后再次通过 AMS 进入系统进程传递 Binder
4. 系统进程得到 Binder ，然后通过调用用户进程曾提供的 ISC 实例，将 Binder 传递给用户进程
5. 用户进程中通过 ServiceConnection 的方法回调，将 Binder 传递给请求绑定服务的 Activity 

### 类角度

1. ContextImpl：请求发起者，对请求参数作基本封装
2. ServiceConnection：接口，定义了多个回调方法，通过方法回调向调用方传递数据（传递），作为 bindService 的参数
3. IServiceConnection：接口，继承自 IInterface ，用于传递 Binder 实例时候的跨进程（由系统进程进入用户进程），封装了 ServiceConnection
4. ActivityManagerService：系统进程入口，接收任务并移交 ActivityService
5. ActivityService：完成 Service 的相关操作的类，基本的逻辑都在此类中
6. ApplicationThread：用于进入目标进程，并将工作移交 ActivityThread
7. ActivityThread：处理 Service 所在进程中 Service 的一些操作，如生命周期方法回调等，并在得到 Binder 之后通过 AMS 传递 Binder 给用户进程

如图：

![](http://blog-1251826226.coscd.myqcloud.com/bind_service_process_class.jpg)

## ANR

Android 有一个 ANR 机制，就是`application not response`，这个机制的实现依赖于 Handler 机制，大致的过程就是：

1. 在做一个事情之前，使用 Handler 发送一条延时 Message ，比如延时 10s 
2. 这个 Message 本身就相当于地雷一样的东西，一旦 Looper 执行了它，就会引起应用报错，比如弹个框说“应用未响应”
3. 在做完这件事情之后，就还需要使用 Handler 从 MessageQueue 中移除这个 Message ，这样在 10s 还没到的时候移除了 Message ，就不会导致应用报错。

对于 Service 来说，执行启动或者是关闭等操作的时候，都需要设置这样一个机制，使得应用不会一直等下去。Service 的埋雷与拆雷操作，都是在 ActiveServices 中执行的。

Service 有前台和后台之分，他们的 TIMEOUT 是不一样的：

- 对于前台服务，TIMEOUT = 20s
- 对于后台服务，TIMEOUT = 200s

### 埋雷

埋雷操作在 AS.bumpServiceExecutingLocked 方法里面，埋雷的时机主要有6个：

- create
- bind
- start
- bring down unbind
- destroy
- unbind

在 Service 要执行以上操作之前，会先调用此方法给 Service 设置一个超时 Message ，并且会判断前后台，设置不同的超时时间。

`com.android.server.am.ActiveServices`

```java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    
   	...
    
    if (r.executeNesting == 0) {
        
        ...
        
        
        scheduleServiceTimeoutLocked(r.app);
    } else if (r.app != null && fg && !r.app.execServicesFg) {
        r.app.execServicesFg = true;
        scheduleServiceTimeoutLocked(r.app);
    }
    
    ...
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.executingServices.size() == 0 || proc.thread == null) {
        return;
    }
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg,
            proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

### 拆雷

拆雷操作在 AS.serviceDoneExecutingLocked 方法里完成，调用这个方法的时机对应着上面埋雷的事件：

`com.android.server.am.ActiveServices`

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
        boolean finishing) {
    
    ...
    
 	mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
            
    ...
}
```

