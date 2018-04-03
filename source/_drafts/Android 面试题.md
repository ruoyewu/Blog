# Android 面试题目

## Android 的四大组件

1.  Activity

    Android 中用户与应用交互的媒介，最基本、最不可缺少的一部分，向用户展示数据、生命周期的管理、逻辑跳转等。

2.  Service

    服务于 Activity ，通常用来后台执行一项比较耗时的操作。

3.  Content Provider

    Android 提供的第三方应用数据的访问方案，可以对外提供数据，类似于数据库的操作，对外透明。

4.  Broadcast Receiver

    接收 Intent 作为触发事件。

## 常用五种布局

1.  FrameLayout
2.  LinearLayout
3.  RelativeLayout
4.  TableLayout
5.  AbsoluteLayout

## 动画

1.  视图动画

    透明度、旋转、缩放、平移等。只能改变 View 展示的效果，不改变 View 本身的参数，所见非所得。

2.  属性动画

    通过插值等方法连续改变 View 的某些属性来完成动画效果，所见即所得。

3.  帧动画

    通过连续播放一连串的图片形成动画效果。

## XML 解析

1.  SAX
2.  DOM
3.  PULL

## RecyclerView 复用

1.  RecyclerView
2.  RecyclerView.Adapter
3.  RecyclerView.ViewHolder

复用单位为 ViewHolder ，通过`RecyclerView.Adapter.onCreateViewHolder()`创建，通过`RecyclerView.Adapter.onBindViewHolder()`为 ViewHolder 加载数据，作初始化等。对于一个 RecyclerVIew 来说，它真正持有的 ViewHolder 的实例一般是比当前屏幕能显示的最大 ViewHolder 数量多 1 ，然后在滑动的时候，会将隐藏的那个 ViewHolder 放到即将出现的位置，并调用`onBindViewHolder()`对这个 ViewHolder 作初始化，这样就能实现以很少的 ViewHolder 实例展示出比较多的数据项，但是复用有一定的风险，有时一个 ViewHolder 可能会保留上一次用户操作的结果，比如在这个 ViewHolder 处于展示状态的时候对其进行操作改变了其内部 View 的一些属性，如果在`onBindViewHolder()`方法内不重新对这个 ViewHolder 的 View 进行状态还原，它就依然会在别的位置还是显示出这样的界面。

## Android 数据存储

1.  SharedPrefrences 存储

    目录`/data/data/${PackageName}/Shared_Pref`

    常用于存储简单的配置信息，采用「键值」方式存取。

2.  文件存储

    目录`/data/data/${PackageName}/files`

    直接读写文件来完成数据的存取。

3.  SQLite 存储

    目录`/data/data/${PackageName}/database`

    关系型数据库。

4.  ContentProvider 存储

    继承`ContentProvider`类重写其数据存取方法，即可供第三方应用获取数据。

5.  网络存储

    利用网络请求的方式，将数据保存在远程服务器上。

## Activity 启动模式

1.  standard（默认）

    没有任何限制，会自动在当前活动栈的栈顶创建一个实例，并显示出来。

2.  singleTop

    栈顶复用

    -   如果要打开的 Activity 位于当前活动栈栈顶

        调用`onPause()` → `onNewIntent()` → `onResume()`重启 Activity 。

    -   其他情况

        创建一个实例并加入当前栈栈顶。

3.  singleTask

    栈内复用。

    -   活动栈中已存在此 Activity 实例

        首先将这个实例所处的活动栈移到顶部，调用`onPause()` → `onNewIntent()` → `onResume()`重启 Activity 。

        -   此实例处于栈顶

            无额外操作。

        -   不处于栈顶

            依次将当前栈栈顶的 Activity 移除栈，直到当前实例处于栈顶。

    -   活动栈中没有此 Activity 实例

        创建一个实例，并加入到当前栈栈顶。

4.  singleInstance

    单一实例，只能独处于一个活动栈。

    -   以这种方式打开 Activity

        -   如果活动栈中已有此 Activity 实例

            将这个实例所处的活动栈移到顶部，调用`onPause()` → `onNewIntent()` → `onResume()`重启 Activity 。

        -   如果活动栈中没有此 Activity 实例

            新建一个活动栈，将这个活动栈移到顶部，新建一个 Activity 实例加入到这个栈的栈顶。

    -   在这样的 Activity 打开其他 Activity

        因为这样的 Activity 只能独处于一个活动栈，所以不管以任何方式打开其他 Activity ，都会重新开辟一个栈，并将这个新的栈放到顶部，新建一个实例加入这个栈栈顶。

一个应用中存放 Activity 是以一个一个活动栈的方式存放的，同时可以有多个活动栈，所以也可以理解为这是一个「二维栈」的结构，第一维以「活动栈」为单位，第二维以「活动」为单位，要想使一个活动处于前台可展示的状态，就需要满足两个条件：

1.  这个活动处于他所在的活动栈的栈顶
2.  这个活动所在的栈也在所有活动栈的顶端

之所以是这种结构，是因为在以`singleTask`和`singleInstance`方式启动 Activity 的时候，有可能会移动一个活动栈的位置。

## Activity 生命周期

1.  onCreate

    创建一个实例

2.  onStart

    不可见 → 可见

3.  onRestart

    不可见 → 可见（在`onStart()`之前调用）

4.  onResume

    非前台 → 前台

5.  onPause

    前台 → 非前台

6.  onStop

    可见 → 不可见

7.  onDestory

    销毁一个实例

## Activity 屏幕旋转生命周期 

1.  不设置`android:configChanges`

    -   横屏 → 竖屏

        各个生命周期 2 次

    -   竖屏 → 横屏

        各个生命周期 1 次

2.  设置`android:configChanges="orientation"`

    各个生命周期 1 次

3. 设置`android:configChanges="orientation|keyboardHidden"`

    不调用生命周期，调用`onConfigurationChanged()`

4.  设置`android:configChanges="orientation|keyboardHidden|screenSize"`

## Service 启动方法

Service 也是 Android 的四大组件之一，可以使用`Local Service`和`Remote Service`两种，都可以用来处理一些不适合放在主进程/主线程处理的工作，并将处理结果反馈给主进程/主线程。

1.  继承`Service`

    继承`Service`类，实现其中的`onBind()`方法。

2.  在`AndroidManifest.xml`中注册

3.  使用`Context.startService()`、`Context.bindService()`调用

    1.  `startService()`

        通过这种方法启动，Service 就会在后台独立运行，不随调用方的销毁而销毁，需要手动调用`stopService()`或`stopSelf()`方法关闭 Service。调用这个方法之后，系统会随之调用`onStartCommand()`方法，调用方与 Service 之间通过`Intent`传递数据。

    2.  `bindService()`

        通过这种方法启动，Service 的生命周期就会与调用方一致，当所有与之绑定的调用方都销毁或解绑之后，Service 就会自动销毁。使用这种方法时，调用方可以与 Service 进行交互。

Service 有很多继承类，实现了不同的功能，如`IntentService`默认将操作放在异步线程中进行，只需要在`onHandleIntent()`方法中执行操作就可以了。

## 注册广播的方式

1.  常驻型广播

    在`AndroidManifest.xml`中注册，理论上生命周期长于应用的生命周期，即应用关闭也能收到广播。

2.  非常驻型广播

    在代码中注册，生命周期与开启这个广播接收器的组件一致。

## 单线程中 Message Handler MessageQueue Looper 关系

Handler 用于获取当前线程的 Looper 对象，Looper 存放着从 MessageQueue 中取出的 Message ，并由 Handler 进行 Message 的转发与处理。

当使用 Handler 发送一条 Message 的时候，这条Message 会首先进入 MessageQueue ，然后在`Looper.loop()`方法里再调用 Handler 处理这条 Message ，这一流程的目的就是将任意一个线程的 Message 发送到 Handler 所在的线程再进行处理。

## 简单解释 Activity Intent IntentFilter Service Broadcast BroadcastReceiver

Intent 是对一个动作的抽象描述，可以使用`Context.startActivity()`启动一个 Activity，使用`Context.sendBroadcast()`发送一个广播，使用`Context.startService()`启动一个服务等。Intent 指定了请求的操作和带操作数据的 Uri 。Intent 可以显式指定某一个执行操作的 Component ，或者也可以隐式指定某一类型执行操作的 Component ，这时需要与 IntentFilter 协作使用。

IntentFilter 在`AndroidManifest.xml`中声明，用于指定它可以操作的 Intent ，然后再使用 Intent 的时候，就能根据 IntentFilter 找到对应的 Component 。

## 简单解释 MVC MVP

Android 应用开发架构，用于解耦合。

### MVC

`model view control`

1.  model

    应用程序的主体，所有的业务逻辑所在

2.  view

    应用程序的用户界面，负责与用户交互，接受用户输入，并显示结果

3.  control

    联系 view 和 model ，一方响应 view 层用户触发的事件，并调用 model 层的逻辑处理相关业务，得到结果再次反馈给 view 层。

通常将 Activity 作为 control 层，布局文件 xml 作为 view 层，其他的类似于数据库读取、网络请求的操作为 model 层，各司其职。但是通常情况下 Activity 并不只是单独地作为 control 层，除了要处理用户的输入外，还要负责一部分 view 层的工作，即改变视图等，这样仍然会使 Activity 中的代码量不断增加，所以后来又出现了 MVP 架构。

### MVP

`model view presenter`

1.  model

    为 presenter 提供接口。

2.  view

    不直接依赖于 model ，持有 presenter 实例。

3.  presenter

    presenter 作为 model 和 view 的中继站，一方面依赖于 model 层提供的接口，另一方面持有 view 的实例，然后当 view 层接收到用户输入的时候，调用 presenter 中对应的方法， presneter 调用 model 层提供的接口，得到结果之后，再通过 view 的实例调用其方法完成数据的反馈。

## 简单解释 ANR

`Application Not Responding`

Android 中有「活动管理器」和「窗口管理器」两个系统服务监视应用程序的响应，在「应用程序 5 s 内没有做出反应」或「BroadcastReceiver 10 s 内没有执行完毕」的时候，就会触发程序无响应对话框。

避免的方法就是，不要在主线程执行可能会耗时的操作，如大量计算、网络请求、文件操作等，如果遇到这种情况，最好在子线程里面进行。

## 简单解释 Force Close

程序出现了异常并且没有进行捕获处理。如`NullPointerException`。

## 描述 Android 系统架构

Android 系统结构分为四层，分别是：

1.  应用程序 (Application)
2.  应用程序框架 (Application Frameworks)
3.  系统运行库与 Android 运行环境 (Libraris & Android Runtime)
4.  Linux 内核 (Linux Kernel)

![](http://blog-1251826226.coscd.myqcloud.com/android-system-architecture.jpg)

### 应用程序

一系列核心应用程序包，如短信、日历、浏览器、联系人等最基础的应用程序，使用 Java 编写，也是一般开发人员所处层次。

### 应用程序框架

提供给开发者的各种 API ，用于进行 APP 快速开发，如：

-   各种基础控件，进行 Android 界面开发
-   内容提供器，访问第三方应用数据
-   资源管理器，访问非代码形式的资源，如图片、视频
-   通知管理器，在状态栏显示通知
-   活动管理器，管理 Activity 生命周期

### 系统运行库和 Android 运行环境

#### 系统运行库

一些 C/C++ 库，供 Android 系统使用，并通过 Android 应用程序框架供开发者使用。

-   Bionic 系统 C 库

    从 BSD 继承的标准 C 系统函数库

-   媒体库

    基于 PacketVideo OpenCORE，支持多种常用音频、视频格式的回放和录制，支持静态的图像文件。编码格式有`MPEG4 H.264 MP3 AAC JPG AMR PNG`

-   Surface Manager

    显示子系统的管理，为多个应用程序提供 2D 和 3D 的无缝融合

-   Webkit LibWebCore

    Web 浏览器引擎

-   SGL

    底层 2D 图形引擎

-   3D Libraries

    基于 OpenGL ES 1.0 APIs ，可以使用硬件 3D 加速或者高度优化的 3D 软加速

-   FreeType

    位图和矢量字体的显示

-   SQLite

    轻型关系数据库

-   其他

#### Android 运行环境

提供 Java 编程语言核心库大多数的功能。

Android 系统会为每个应用程序构建一个 Dalvik 虚拟机实例，这个虚拟式能够直接执行`.dex`文件

### Linux 内核

Android 核心系统服务依赖于 Linux 2.6 内核，大多都是继承了 Linux 的方法，但对以下两个做了修改：

-   IPC 机制

    用于进程间通信。

-   电源管理

    优化电量管理，更加省电。

## 描述 ContentProvider 如何共享数据

应用程序通过实现 ContentProvider 的接口对外提供数据，并且是以类似数据库中表的方式将数据暴露出去的，它通过类似于数据库的`query()`、`insert()`、`delete()`、`update()`方法对外提供数据操作，并且能够被所有应用访问。

ContentProvider 数据共享的基本原理是进程间通信，通过 ContentResolver 完成，ContentResolver 根据提供的 Uri 得到不同进程的对应的 ContentProvider 提供的数据之后，再通过 Binder 传输到本进程，实现进程间信息的传递。

## Service 与 Thread 区别

1.  Service

    Service 是系统组件，由系统进程托管，他们之间的通信类似于 Client 和 Server ，是一种轻量级的 IPC 通信，通信的载体是 Binder 。

2.  Thread

    由本应用程序托管，是程序执行的最小单元，用来执行一些异步操作。

Service 可以分为 Local Service 和 Remote Service ，Local Service 运行在主进程的 main 线程上，而Remote Service 则是运行在独立进程的 main 线程上。

Thread 独立于 Activity 存在，当这个 Activity 被 finish 之后，就不能再持有 Thread 的引用，无法对其进行控制，而通过 Service ，每个 Activity 都可以通过一定的方法联系到 Service ，如`Context.startService()`、`Context.bindService()`等对这里面运行的 Thread 进行控制。

## Android 抛出 Runtime 异常原因

如`NullPointerException`

## IntentService 优点

继承于 Servcice ，自动在其中创建一个 Thread 执行操作，只需要实现`onHandleIntent()`方法即可。

## 保存 Activity 的状态

`onSaveInstanceState()`和`onRestoreInstanceState()`

## Activity 退出的方法

1.  单一 Activity

    直接调用`finish()`或者类似方法

2.  多 Activity

    1.  使用 ActivityManager 记录每一个 Activity ，需要的时候逐个关闭
    2.  发送广播，指定的 Activity 接收到广播后会关闭

## 简单介绍 AIDL

`Android Interface Define Language`

主要用于进程间信息传递。

## Android 运行时权限与文件系统权限

### 运行时权限

与对应的运行文件所具有的权限无关，由 Android 系统对这个权限进行管理。

1.  APK 必须签名
2.  基于 UserID 的进程级别的安全机制
3.  默认 APK 生成的数据对外不可见
4.  AndroidManifest.xml 中显式权限声明
5.  权限继承/ UserID 继承

### 文件系统权限

由 Linux 内核授权，涉及到文件是否可以被某个用户访问。

## Android 系统的优势与不足

1.  优势
    1.  平台开放性
    2.  丰富硬件
    3.  开发无限制
    4.  Google 应用
2.  不足
    1.  安全与隐私

## Android DVM 进程、Linux 进程、应用程序进程是否是同一概念

DVM 即 Dalvik 虚拟机，每个应用程序都拥有自己的一个进程，而每个应用程序都在一个独立的 Dalvik 虚拟机实例中运行，每个 DVM 又都在一个 Linux 进程中，所以可以说是同一个概念。

## JNI

`java native interface`Java 本地接口

能够在 Java 类中引用本地其他类型语言（如 C/C++ ）编写的代码，Java 平台的东西，与 Android 无关，

## IPC 机制

1.  Bundle

    Bundle 实现了`Parcelable`序列化，所以可以方便地在进程间传递。如在一个 Activity 中启动另一个进程中的 Activity 的时候，就可以通过 Bundle 传递信息给另一进程的 Activity 。

2.  文件共享

    两个进程通过读/写同一个文件完成数据的交互，虽然有可能在多个进程同时对一个文件进行写操作的时候出现问题，但是只要控制得当，还是可以使用的。

3.  Messenger

    轻量级 IPC 方案，底层实现是 AIDL 。

    1.  继承一个 Service ，得到自己的 Service
    2.  构建一个自定义类继承 Handler 类重写`handleMessage()`方法，在这里面接收到客户端发来的 Message 并作出相应处理
    3.  使用 Handler 实例构建一个 Messenger 实例，并通过`Messenger.getBinder()`得到 Binder 实例，在 Service 的`onBind()`方法中返回
    4.  在 Activity 中构建一个 ServiceConnection 实例，实现`onServiceConnected(ComponentName, IBinder)`和`onServiceDisconnected()`方法，在前一个方法中通过它提供的 IBinder 构建对应的 Messenger 实例，并通过`Messenger.send(Message)`方法发送一个 Message 给 Service ，完成信息的发送
    5.  如果需要从 Service 到 Activity 信息的传送，需要按照上面的方法完成一个反向的功能即可

4.  AIDL

    通过构建一个 AIDL 接口，规定能够用来传递信息的方法，然后在 Service 中实现这个接口中对应的方法，在 Activity 中通过 ServiceConnection 中的`onServiceConnected()`方法得到这个接口对应的实例，通过调用这个接口的方法，就可以完成与 Service 的通信。

    1.  AIDL 提供的直接能够传输的数据类型包括`int long boolean float double String`，其他的所有数据类型都要通过实现 Parcelable 接口是指可以参与进程间传递。
    2.  首先要编写 AIDL 接口文件，如果需要用到其他数据类型，还需要额外添加一个 AIDL 文件，用来描述对应的 Java 数据类型，并且还要编写一个对应的 Java 类实现 Parcelable 接口。
    3.  在 Service 中通过实例化 AIDL 接口生成的 JAVA 接口中的`${接口}.Stub`这个类得到对应的 Binder 实例，然后在`onBind()`方法中返回这个实例
    4.  在 Activity 中通过`bindService()`方法得到 Service 中返回的 Binder 实例，然后通过`${接口}.Stub.asInterface()`方法得到定义的 AIDL 接口对应的实例，然后就可以通过调用这个接口中的方法完成 Activity 与 Service 之间的通信了。

5.  ContentProvider

    通过继承 ContentProvider 类并实现其中的增删改查方法完成数据提供，然后再通过 ContentResolver 得到需要的 ContentProvider 然后调用它的各个方法完成数据上的交互。

6.  Socket

    通过 Socket 的方式传递数据，即网络通信，分别在两个进程中构建一个服务器和客户端，客户端通过向服务端发送请求的方式与之进行数据交互。

## NDK

`native development kit`

一系列工具的集合，帮助开发者使用 C/C++ 得动态库，并自动将 so 和 java 应用打包成 APK 。交叉编译。Android 平台的东西。

使用 C/C++ 作为应用的开发语言，能够对 C/C++ 语言进行编译，并可打包成`.so·`库，方便复用。使用 C/C++ 开发应用，应用的反编译能力增强，性能提高。

## 热修复原理

### 利用 Java 加载机制

Tinker QQ空间超级补丁

Android 运行 Java 代码使用的是 Dalvik 虚拟机运行`.dex`文件，加载的流程一般为：

1. 加载`.dex`文件，将所有的`.dex`文件保存起来
2. 需要使用某个类的时候，通过`BaseDexClassLoader.findClass(String)`找到对应的类，这个寻找的过程就是遍历`.dex`类，找到就会返回这个类，停止寻找，否则就返回 null

由此可以知道，不管`.dex`的文件中有多少个类，只要 BaseDexClassLoader 找到一个类之后，就会停止，所以当需要调换一个类的时候，只需要使加载器先加载到修改过的类就好了，根据这个原理，如果一个类有问题，可以将修改好的类重新打包成`.dex`文件，并将这个文件放到有 bug 的类所在的`.dex`文件的前面，即使类加载器搜索类的时候，先找到的是这个类，就可以完成动态类的替换了。但是加载

### 修改底层 C 代码

andfix 

通过修改底层的 C 语言来修改，当在 Java 中调用方法的时候，在 C 语言的层面来说，就是使用指针指向这个 Java 方法对应的位置，所以如果某个类的某个方法出错了，只需要重新写一个正确的方法，然后通过修改 C 使指向之前的方法的指针指向新的方法的位置，就可以完成热修复。相比较通过 Java 的类加载机制来实现热修复来说，这种方法可以即时生效。

## View 绘制流程

View 的绘制流程与事件分发，都是从最底层的 View 将一个信息一层一层传递到最顶层，从而完成某种功能的过程。对于一个 View 的绘制过程来说，包括 measure 、 layout 、 draw 三个部分，分别就是测算 View 所占大小、为 View 分配布局空间、画出 View 的内容。

对于每一个 View 来说，都是要按照这三个流程一步一步走，最终显示出来，对于一个界面来说，则是从最低层的 View 层层调用自己和子 View 的这三个方法，完成一个页面的正常显示。

调用一个页面的 View 进行重绘的逻辑在` ViewRootImpl.performTraversals()`方法中，在这个方法中，依次调用了`performMeasure()`、`performLayout()`、`performDraw()`这三个方法，三个方法内部调用的分别就是根 View 的三个相关方法。

### Measure

在`ViewRootImpl.performMeasure()`方法里调用根 View 的`measure(int, int)`方法，然后调用`View.onMeasure(int, int)`方法设置测算后的 View 的尺寸，一般情况下如果是 ViewGroup ，会重写这个方法，在里面调用子 View 的`measure(int, int)`方法逐层完成测量。

### Layout

在`ViewRootImpl.performLayout()`方法中调用`View.layout(int, int, int, int)`方法，设置 View 的位置，并调用`onLayout()`方法，供自定义 ViewGroup 的时候重写以调用子 View 的 layout 方法。

### Draw

在`ViewRootImpl.performDraw()`方法中通过多层调用到`View.draw(Canvas)`方法，用于实现具体的 View 的绘制功能，分为六步：

> 1. Draw the background
> 2. If necessary, save the canvas's layers to prepare for fading
> 3. Draw view's content
> 4. Draw children
> 5. If necessary, draw the fading edegs and restore layers
> 6. Draw decorations (scrollbars for instance)

当一个 View 有自身的内容，比如一个 TextView ，就需要实现`onDraw(Canvas)`方法来对自身的内容（对 TextView 来说就是文字内容）进行绘制，同时通过`dispatchDraw(Canvas)`用来调用子 View 的绘制方法，一般的 View 对这个方法是一个空实现，ViewGroup 对这个方法进行了重写，逐个调用了子 View 的`draw(Canvas)`方法。

## 事件分发

Android 事件分发机制，是指当用户触摸了显示屏之后，这一触摸事件在 Android 中一整个处理的过程，而 Android 的事件分发流程就是一个从最顶级的 Activity 一直传递到最低级的 View 的一个过程，了解事件分发的主要目的还是方面我们使用自定义的 ViewGroup ，是一个 ViewGroup 能够对用户的输入事件作出正确的响应，所以最主要的是了解一个时间，从最顶级的 ViewGroup 往下传递的时候，是怎么传递的，以及这个传递过程中的规则、方法与对事件的处理。

### 分发流程

`Activity → ViewGroup → ... → ViewGroup → View`

事件的分发采用的是`dispatchTouchEvent(MotionEvent)`这个方法，这个方法在 View 和 ViewGroup 中实现的方法不同，在 View 中，由于 View 是这个分发流程的最底层，所以这个方法里会调用自己的`onTouchEvent(MotionEvent)` 方法，返回自己是否消费了这个事件的结果，并传给自己的父控件。在 ViewGroup 中，首先会调用自己的`onInterceptTouchEvent(MotionEvent)`方法，这个方法默认返回 false ，自定义的 ViewGroup 需要重写这个方法以构建自己的拦截方式，如果拦截的话，就会经过一定的调用，最终调用自己的`onTouchEvent()`方法，ViewGroup 中的这个方法是直接从 View 中继承过来的，然后将自己的`onTouchEvent()`方法的返回结果反馈给自己的 父控件，或者直接就是 Activity（如果自己就是最顶层 ViewGroup），如果自己表示不拦截，就会逐个遍历自己的子控件，调用其`dispatchTouchEvent()`方法，然后将其返回结果反馈给上层控件。

### 响应流程

`View → ViewGroup → ... → ViewGroup → Activity` 

每个 ViewGroup 接收到一系列事件的时候，都会调用`onInterceptTouchEvent()`方法判断是否拦截，即是否再往下分发这个事件，但是是否消费还是要通过`onTouchEvent()`的返回结果来确定，就是说，即便一个事件被一个 ViewGroup 拦截了，但是它没有消费这个事件，那么这个事件还会继续向上传递，这个 ViewGroup 的上层控件还是可以消费这个事件的。

## 图片异步加载缓存方案

在使用 RecyclerView 的时候，有可能需要在一个列表中加载多张图片，由于网络延迟以及 RecyclerView 的重用机制，有可能会出现一个 ImageView 上加载的图片并不是它应该显示的那一张。

## Android 内存泄漏及管理

由于 Java 本身具有的垃圾回收机制，使得程序员可以不用考虑对内存的释放，但是一旦 Java 本身不能回收某些实际上已经不再使用的数据，就会造成可用内存逐渐降低，直到程序崩溃。

### 内存泄漏原因

1. Static Activities

   静态变量的周期一般长于一个 Activity 的生命周期，所以当一个 Activity 被强引用的时候，Java 无法回收这个 Activity ，但是如果使用弱引用，就不会阻止回收

   使用 WeakReference 持有 Activity 实例

2. Static Views

   使用弱引用

3. Inner Classes

   使用静态内部类

4. Anonymous Classes

   匿名内部类都会隐式持有外部类的引用，如果这个类的生命周期比 Activity 长，就会导致 Activity 无法回收

   不实用匿名内部类，将其转化为静态内部类

5. Handler

   不实用匿名内部类，将其转化为静态内部类

6. Threads

   不实用匿名内部类，将其转化为静态内部类

7. TimerTask

   不实用匿名内部类，将其转化为静态内部类

8. Sensor Manager

   设置监听也是同样，事件的分发端会持有当前监听器的引用，如果监听器是 Activity ，那么这个 Activity 就永远无法回收，所以需要在销毁阶段手动解除监听。

   在 Activity 的`onDestroy()`周期取消监听等

## Activity 与 Fragment 通信

### Activity → Fragment

1. 通过 Bundle

   ```java
   Fragment fragment = new DemoFragment();
   Bundle bundle = new Bundle();
   // 为 Bundle 添加数据
   fragment.setArguments(bundle);
   ```

2. 使用接口

   使 Fragment 实现某个接口，然后在 Activity 中持有这些 Fragment 的实例，需要的时候直接调用 Fragment 实现的接口方法

3. 使用广播

### Fragment → Activity

1. 使用接口

   Activity 实现某个接口，然后在 Fragment 中通过`getActivity()`得到 Fragment 所在 Activity 并强制转化为这个接口类型，然后调用接口的方法

2. 直接引用

   在 Fragment 中得到 Activity 实例并强制转化到对应的 Activity 类，然后直接调用 Activity 方法

3. 广播

### Fragment → Fragment

1. Activity 作中介
   1. 在 Fragment 中先通过与 Activity 通信得到另一个 Fragment 实例，然后强制转换调用另一个 Fragment 的方法
   2. 在 Frament 中与 Activity 通信，然后由 Activity 与另一个 Fragment 通信
2. 广播

## 六大原则

1. 单一职责原则

   一个类只负责一项职责

2. 里氏替换原则

   子类能够无条件替换父类（子类只拓展，不修改）

3. 依赖倒置原则

   高层不依赖低层，二者都依赖抽象

   细节依赖抽象

4. 接口隔离原则

   只向使用者开发使用者需要的方法

5. 迪米特法则

   低耦合，只对其他类由最少的了解

6. 开闭原则

   对扩展开放，对修改封闭

## 设计模式

1. 单例模式
2. Builder 模式
3. 原型模式
4. 工厂方法模式
5. 抽象工厂模式
6. 策略模式
7. 状态模式
8. 责任链模式
9. 解释器模式
10. 命令模式
11. 观察者模式
12. 备忘录模式
13. 迭代器模式
14. 模版方法模式
15. 访问者模式
16. 中介者模式
17. 代理模式
18. 组合模式
19. 适配器模式
20. 装饰模式
21. 享元模式
22. 外观模式
23. 桥接模式

## SQLite 数据库与 APK 一起发布

## Intent 使用

Intent 作为 Android 四大组件之间的通信工具，在一个组件中启动另一个组件或者是给另一个组件发消息，一般使用的都是 Intent ，将 Intent 需要执行的逻辑放到内部，并可以传递一些数据，就可以通过startActivity(Intent)、startService(Intent)等方法。

## Service 生命周期

Service 根据其不同的调用方式拥有不同的周期。

1. start
   1. `startService()`、`stopService()`
   2. `onCreate` 、`onStart()`、`onStartCommand()`、`onDestroy()`
2. bind
   1. `bindService()`、`unbindService()`
   2. `onBind()`、`onUnbind()`、`onDestory`

## Android 应用启动过程

1. Launcher 通过 Binder 进程间通信机制通知 ActivityManagerService 启动一个 Activity
2. AMS 通过 Binder 通知 Launcher 进入 Paused 状态
3. Launcher 通过 Binder 通知 AMS 已进入 Paused 状态，AMS 创建一个新的进程，启动一个 ActivityThread 实例
4. ActivityThread 通过 Binder 将 ApplicationThread 实例传给 AMS 用于两者之间的通信
5. AMS 通过 Binder 通知 ActivityThread 准备就绪，于是 AT 启动了一个 Activity

## 非 UI 线程更新 UI

访问 UI 的时候调用的是`ViewRootImpl.checkThread()`方法检测是否属于 UI 线程，所以可以在 ViewRootImpl 还未创建的时候更新 UI ，因为 ViewRootImpl 在`onResume()`周期才会创建。

另外，所谓的只能在 UI 线程中更新 UI ，只是一个一般的说法，正确的说法应该是，只能在 ViewRootImp 与 RootView 被创建的线程中更新 UI ：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

这一句`mThread != Thread.currentThread()`就表明了，只要当前的更新 UI 的操作与`mThread`是同一个线程就行，而这个`mThread`就是 ViewRootImpl 被创建的时候调用的`mThread = Thread.currentThread()`来获取的，不过一般情况下这些 UI 操作都是默认的都存在于 UI 线程，如果想要在其他线程更新 UI 而不报错，只需要在这个线程中拥有自己的 ViewRootImpl 就好了。

## LruCache

内部使用 LinkedHashMap 存储数据，使用`LRU(Least Recently Used) : 近期最少使用`算法。使用的时候需要指定最大缓存容量，然后重写`sizeOf(K, V)`方法即可。

```java
LruCache lruCache = new LruCache(MAX_SIZE) {
    @Override
    protected int sizeOf(K key, V value) {
        return SIZE_OF(key, value);
    }
}
```

如上，需要根据 key 和 value 确定这个数据的大小，然后指定一个最大容量，在使用的时候就可以将 map 所存的数据的大小维持在 MAX_SIZE 以内，每次访问一个数据或添加一个数据的时候，会将这个数据移到头部，当所存的大小大于 MAZ_SIZE 的时候，会删除尾部的数据。

# Java 面试题目

## transient 使用

**关键字标记**	防止被标记变量参与序列化

不过这只是一个标记性关键字，如果在看到这段代码的时候，阅读者会知道这个变量不参与序列化，但是如果在序列化的代码实现中依旧序列化了这个变量，它依旧是可以被序列化的，在 Android Studio 上有默认的序列化工具，对一个类序列化的时候，被标记`transient`的变量，将会自动不加入序列化。

## GC

`Gabage Collection (垃圾收集器)`

Java 使用的是垃圾回收机制，可以自动完成不再使用的实例的内存释放工作，不需要编程人员手动释放内存，方便开发， GC 的工作原理是**自动监测对象是否超出了作用域而达到自动回收内存的目的**，Java 没有提供释放对象内存的显示操作，如果要防止 GC 无法完成的内存回收，可以在不需要使用的时候手动使其等于`null`，使对象不可达，这时 GC 就会回收这部分内存。另外，GC 并不是一直在执行，而是隔一段时间或者是堆区的容量不多的时候，就执行一次垃圾回收，将那些不再使用的对象释放掉，从而增加可用空间。

### 垃圾回收算法

1. 引用计数法

   当一个对象被创建的时候，置计数值为 1 ，每当一个对象被引用一次，计数器就会 +1 ，每当超出生命周期或者被设置成新值，就会 -1 ，当计数值为 0 的时候，就会被回收

   无法解决循环引用，但是执行简单

2. tracing 算法（标记-清除算法）

   1. 根搜索算法

      根据一个**GC Root**对象作为根对象，以对象之间的引用关系作为边，以其他对象作为节点，判断从**GC Root**到其他节点的可达性，如果可达，即表示无需回收，否则则需要回收

      可作为 GC Root 的对象：

      1. 虚拟机栈中引用的对象（本地变量表）
      2. 方法区中静态属性的对象
      3. 方法区中常量引用的对象
      4. 本地方法栈中引用的对象（Native 对象）

   2. tracing 算法

      从根对象集合开始，对每一个可达对象做一个标记，然后扫描一遍，对所有没有标记的对象作回收处理

   3. compacting 算法（标记-整理算法）

      tracing 算法只对被标记的对象作了回收，这样会出现很多内存碎片，compacting 算法就是在 tracing 算法的基础上再做整理，将使用中的内存做一个整合，然后将空间区整合到一起

   4. copying 算法 （Compacting Collector）

      将内存分为两个区域，即**空闲面**和**对象面**，新建的对象都会添加到对象面，空闲面是一个空闲区，当检测到对象面空间不够的时候，就将对象面被标记的对象全部复制到空闲面，然后将对象面清空，再将两面反转

   5. generation 算法（Generational Collector）

      根据对象的生命周期将其分为**年轻代**、**年老代**、**持久代**，不同的分类的回收机制不一样

      #### 年轻代（Young Generation）

      年轻代内存会按照一定比例划分为多个区，如按照 8:1:1 的比例划分为`eden : survivor0 : survivor1`

      1. 首先，最初创建的对象会保存在 eden 区，当执行回收的时候，将 eden 区标记的对象放到 survivor0 区，然后清空 eden ，当 survivor0 也满的时候，就将 eden 和 survivor0 中的被标记对象放到 survivor1 中，然后清空 eden 和 survivor0 ，如此重复
      2. 当 survivor1 中也满的时候，就会将 survivor1 中被标记的对象放到年老代，然后清空 survivor 1

      年轻代的 GC 叫做 Minor GC ，发生频率较高

      #### 年老代（Old Generation）

      年老代中的对象一般都是从 survivor1 中存活下来的对象，所以生命周期比较长，内存大小比年轻代大，执行的 GC 为 Major GC 即 Full GC

      #### 持久代（Permanent Generation）

      一般情况下不会 GC 的对象，如静态对象、生命周期是从程序开始到结束的对象等

## Switch 作用类型

`byte char short int String(after JDK1.7)`

## 构造方法

**不能被重写，能被重载**

## 面对对象特征

1. 继承

   一个类可以继承另一个类

2. 封装

   功能以类划分，并且类内部对外界透明

3. 多态

   一个方法可以有多种实现

4. 抽象

   通过构建接口、抽象类的方法，使别的类依赖于抽象而不是具体的某个人，然后使用的时候，是使用这个抽象的具体实现，同样可以有多种实现，而不需要依赖于某一个具体类