---
title: Android Service 启动流程
date: 2018-05-09 21:40
tags:
	- android
---

## 开启服务流程

### 方法调用过程

#### ContextImpl.startService

设置所要开启的服务的基本配置和信息，类型、是否前台等。通过 Binder 调用 AMS.startService 方法，进入 system 进程。

#### AMS.startService

判断 Intent 和调用方包名可用性，获得当前进程 uid 和 pid ，调用 AS.startServiceLocked ，参数增加。

#### AS.startServiceLocked

判断调用方为前台或后台，调用 AS.retrieveServiceLocked 得到 ServiceLookupResult ，进而得到 ServiceRecord （没有时会创建）。

验证调用者，验证权限（会开启权限验证 Activity ），延迟启动处理。调用 AS.startServiceInnerLocked 方法。

#### AS.startServiceInnerLocked

电量记录，调用 AS.bringUpServiceLocked 方法，处理结果。

#### AS.bringUpServiceLocked

判断 Service 是否已经开启，判断是否在等待重启，延迟判断，用户判断，进程判断（如果目标进程没有开启，先去开启进程，调用 AMS.startProcessLocked 方法），调用 AS.realStartServiceLock ed 方法。

#### AS.realStartServiceLocked

调用 AS.bumpServiceExecutingLocked 方法完成解除超时的操作，调用 APT.scheduleCreateService 进入目标进程的 binder 线程，走向 Service.onCreate 方法。

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
3. ActivityServices：服务管理的相关类，实际执行服务的创建、销毁等工作，一般被 AMS 调用相关功能
4. ApplicationThread：运行在目标进程，是一个 Binder 类，用于接收系统进程消息，然后通过 Handler 交由主线程执行工作
5. ActivityThread：进程的主线程所在地，启动主线程 Looper ，并通过 Handler 接收消息然后相应执行，在这里面调用 Service.onCreate 等方法
6. ServiceRecord：表示一个运行的应用服务，继承自 Binder ，存放服务相关信息，在进程切换的过程中随之转移。

如图：

![](http://blog-1251826226.coscd.myqcloud.com/start_service_process_class.jpg)

## ANR

Android 有一个 ANR 机制，就是`application not response`，这个机制的实现依赖于 Handler 机制，大致的过程就是：

1. 在做一个事情之前，使用 Handler 发送一条延时 Message ，比如延时 10s 
2. 这个 Message 本身就相当于地雷一样的东西，一旦 Looper 执行了它，就会引起应用报错，比如弹个框说“应用未响应”
3. 在做完这件事情之后，就还需要使用 Handler 从 MessageQueue 中移除这个 Message ，这样在 10s 还没到的时候移除了 Message ，就不会导致应用报错。

对于 Service 来说，执行启动或者是关闭等操作的时候，都需要设置这样一个机制，使得应用不会一直等下去。Service 的埋雷与拆雷操作，都是在 ActivityServices 中执行的。

Service 有前台和后台之分，他们的 TIMEOUT 是不一样的：

- 对于前台服务，TIMEOUT = 20s
- 对于后台服务，TIMEOUT = 200s

### 埋雷

埋雷操作在 AS.bumpServiceExecutingLocked 方法里面，埋雷的时机主要有6个：

- create
- binde
- start
- bring down unbind
- destroy
- unbind

在 Service 要执行以上操作之前，会先调用此方法给 Service 设置一个超时 Message ，并且会判断前后台，设置不同的超时时间。

`com.android.server.am.ActivityServices`

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

`com.android.server.am.ActivityServices`

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
        boolean finishing) {
    
    ...
    
 	mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
            
    ...
}
```

