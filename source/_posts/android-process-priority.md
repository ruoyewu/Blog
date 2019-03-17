---
title: Android 进程优先级
date: 2019-02-24 22:00
tags:
	- android
---

Android 手机的内存是有限的，一般情况下用户打开了某个软件然后返回到主菜单的时候，应用所占的内存一般不会直接释放，而是先保留一段时间。随着用户打开的应用越来越多，内存总有见底的时候，这时就需要移除某些在系统看来不需要的进程，释放出足够的空间供用户打开新的应用。

先移除哪些进程是需要根据进程的优先级判断的，优先级越高，被杀死的时机也就越晚，下面是一般情况下进程优先级的排序。

## 进程划分

1. 前台进程
2. 可见进程
3. 服务进程
4. 后台进程
5. 空进程·

### 前台进程

1.  与用户交互的 Activity
2.  与前台 Activity 绑定的 Service
3.  调用了`startForeground()`的 Service
4.  正在执行`onCreate()`,`onStart()`, `onDestroy()`的 Activity
5.  包含正在执行`onReceive()`的 BroadcastReceiver

### 可见进程

1.  不处于前台，但依旧可见的 Activity ，如打开了一个对话框，此时用户不可以直接与 Activity 交互，但是这个 Activity 依然为用户可见（已经调用了`onPause()`，但还未调用`onStop()`）
2.  可见 Activity 绑定的 Service

### 服务进程

1.  启动的 Service

### 后台进程

1.  不可见的 Activity （调用了`onStop()`）

### 空进程

1.  没有任何活动的进程