---
title: Android Activity 启动流程
date: 2018-05-11 13:20
tags:
	- android
---

## 启动 ACTIVITY 流程

>缩写：
>
>ActivityManagerService：AMS
>
>Instrumentation： IS
>
>ActivityThread：AT
>
>ActivityStarter：AS
>
>ActivityStackSupervisor：ASS
>
>ActivityStack：AST

### 方法调用过程

#### Activity.startActivity

调用 IS.execStartActivity 并将结果使用 AT.sendActivityResult 分发。

#### IS.execStartActivity

判断是否应该阻止 Activity 的启动，如果不，则调用 AMS.startActivity 进入系统进程。

#### AMS.startActivity

进入系统进程，封装之后调用 AS.startActivityMyWait 。

#### AS.startActivityMayWait

参数众多，梳理一下各个参数及其含义：

> caller: IApplicationThread，调用者进程的 AT ，用于之后的进程通信
>
> callingUid: int，调用者进程 ID
>
> collingPackage: String，调用者应用包名
>
> intent: Intent，启动 Activity 的参数，包含要启动的 Activity 信息
>
> resolvedType: String，intent 的 MIME 数据类型
>
> voiceSession: IVoiceInteractionSession
>
> voiceInteractor: IVoiceInteractor
>
> resultTo: IBinder，调用方 Activity 的 mToken
>
> resultWho: String，调用方 Activity 的 embeddedID
>
> requestCode: int，请求码，用于后续的 onActivityResult
>
> startFlags: int，启动参数
>
> profilterInfo: ProfileterInfo，null
>
> outResult: WaitResult，null
>
> globalConfig: Configuration，null
>
> bOptions: Bundle，启动配置，在 Activity.startActivity 传入
>
> ignoreTargetSecurity: boolean，false
>
> userId: int，调用者 ID
>
> inTask: TaskRecord，null
>
> reason: String，调用此方法的方法名

根据 intent 信息检索对应的 Activity (ASS.resolveIntent、ASS.resolveActivity)，对 heavy-weight 进程作处理，封装，使用系统进程的 Pid 和 Uid 等开启 Activity (调用 AS.startActivityLocked)。

#### AS.startActivityLocked

对 reason 参数作检测并利用，调用 AS.startActivity 。

#### AS.startActivity

验证调用方进程信息，验证用户信息，验证 intent 等，验证开启 Activity 的权限，需要的话开启权限验证 Activity 而不是当前正要启动的 Activity ，然后启动在此之前正在等待启动的 Activity ，完了之后，再启动当前 Activity ，调用 AS.startActivityUnchecked 。

#### AS.startActivityUnchecked

选择启动模式（四种启动模式，对应着 Activity 的复用等），选择接下来的走向，是否开辟新的活动栈，是否重用已存在的 Activity ，找到 Activity 所在的栈（可能会创建一个栈），然后调用 AST.startActivityLocked 和 ASS.resumeFocusedStackTopActivityLocked 。

#### AST.startActivityLocked

在活动栈中为 Activity 创建容器。

#### ASS.resumeFocusedStackTopActivityLocked

判断是否可以显示 Activity ，调用 AST.resumeTopActivityUncheckedLocked 。

#### AST.resumeTopActivityUncheckedLocked

判断是否第一次执行到这里（避免多次执行），调用 AST.resumeTopActivityInnerLocked 。

#### AST.resumeTopActivityInnerLocked

判断系统状态，判断 Activity 状态，判断是否为 Home 应用，判断用户有效性，判断 Activity 是否已经存在，如果存在，则走向调用 Activity.onNewIntent Activity.onResume 等方法，否则调用 ASS.startSpecificActivityLocked 方法创建 Activity 并启动。

#### ASS.startSpecificActivityLocked

判断 Activity 所在进程是否存在，如果不，先走向创建进程（AMS.startProcessLocked），最终仍会调用 ASS.realStartActivityLocked 启动 Activity 。

#### ASS.realStartActivityLocked

调用 APT.scheduleLaunchActivity 进入目标进程启动 Activity 。

#### APT.scheduleLaunchActivity

使用 Handler 进入主线程，调用 AT.handleLaunchActivity 方法。

#### AT.handleLaunchActivity

调用 ComponentCallbacks2.onConfigurationChanged 方法，创建 Activity 实例，创建 Context 实例，调用 Activity.attach 初始化，再逐个调用 Activity.onCreate Activity.onStart 等方法，走向开发者层面，完成创建流程。

### 总体流程

#### 进程角度

1. 首先从用户进程发起请求，通过 Binder 进入系统进程
2. 系统进程完成绝大多数的工作，创建目标进程，创建活动栈等等，完成之后通过 Binder 进入目标进程
3. 在目标进程中完成 Activity 的创建，以及各个生命周期方法的调用等。

#### 类角度

1. Activity：使用 startActivity 发起请求
2. Instrumentation：对 Activity 的生命周期进行管理的工具类，从这里进入系统进程
3. ActivityManagerService：系统进程中各种服务的入口，负责接收任务并分发给 AS
4. ActivityStarter：为启动 Activity 的做判断、准备工作，栈创建、进程创建等
5. ActivityStackSupervisor：提供对活动栈的管理，调用 APT 进入用户进程
6. ActivityStack：活动栈类，负责为 Activity 提供空间
7. ApplicationThread：进入当前进程的媒介，继承自 Binder
8. ActivityThread：主线程所有工作的入口，完成了 Activity 实例创建等工作，Activity 的各生命周期或多或少都由此开启

如图：

![](http://blog-1251826226.coscd.myqcloud.com/start_activity_process_class.jpg)

