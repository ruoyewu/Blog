---
title: Android Context 介绍
date: 2018-04-26 16:00
tags:
	- android
---

## Context

Context 是一个抽象类，根据其翻译“上下文、环境”就可以了解到，Context 与应用的运行密不可分。下面是源码中对 Context 的介绍：

> Interface to global information about an application environment.  This is an abstract class whose implementation is provided by the Android system. It allows access to application-specific resources and classes, as well asup-calls for application-level operations such as launching activities, broadcasting and receiving intents, etc.

下面是 Context 类的继承结构：

![context_tree](http://blog-1251826226.coscd.myqcloud.com/context_tree.jpg)

首先 ContextImpl 是 Context 的具体的实现类，但是其他类在使用 ContextImpl 的时候都没有直接使用这个类，而是通过一个对 ContextImpl 的封装类 ContextWrapper 完成对 Context 功能的调用，比如 Activity 、 Application 、 Service 等都是通过直接或间接的方式继承了 ContextWrapper 。

### 为什么使用 ContextWrapper

ContextWrapper 继承了 Context 类，但是 ContextWrapper 中所有方法的实现都是转而调用 ContextImpl 类的方法来完成的，如：

```java
public class ContextWrapper extends Context {
    Context mBase;

    ...
    
    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }
    
    ...
}
```

这里的 mBase 就是 ContextImpl 实例，在 ContextWrapper 中持有一个 Context 实例，并通过调用这个实例的方法的方式完成自身的功能，这里用到了一种设计模式，叫「装饰器模式」。

#### 装饰者模式 VS 继承机制

装饰器的作用是**动态地给现有的类添加新功能**，但是如果只是为了给类增加新功能的话，基于继承的机制也能完成这种功能，但是一个问题是，ContextImpl 只是 Context 的一个实现类，以后说不定还会出现其他的 Conetxt 实现类，在这种情况下，那么那些 Activity 、 Application 等是否要更换父类或者是增加一个继承于新的实现类的子类，那么这样显然会增加不必要的开销。所以就有了一个用于装饰 Context 某个实例的 ContextWrapper ，ContextWrapper 可以通过更改构造参数的方式随时切换不同的 Context 实现类，然后使用 ContextWrapper 作为 Activity 等的父类，对于子类看来与直接继承 ContextImpl 没有什么区别，如果要增加新功能或者加一些限制什么的只需要重写 ContextWrapper 的方法就好了。对于 Activity 和 Service 来说，它们身为不同的组件，在某些方面肯定会有些不一样，就可以通过重写 ContextWrapper 的方法实现特定的功能，而不会对 ContextImpl 这个类有任何的影响。

### Context 数量

由 Context 的继承结构，知道所有的 Application 、 Activity 、 Service 都是一个 Context ，实际上也只有这些继承于 Context ，其他地方如 ContentProvider 使用的 Context 都直接或间接来源于以上三个 Context 子类，所以一个应用中 Context 的数量为`Activity数量 + service数量 + 1(Application)`。

在多进程应用中，Application 使用的仍然是同一个，只是会重复调用`Application:onCreate()`方法。

## Context 子类

Context 的三个子类 Application 、 Activity 、 Service 虽然都继承于 ContextWrapper ，但是功能会略有不一，一般是通过重写 ContextWrapper 方法或者间接继承 ContextWrapper 实现功能的差异，而它们之间的差异如下：

`C(条件) Y(可以) N(不可以)`

|          功能          | Application | Activity | Service |
| :--------------------: | :---------: | :------: | :-----: |
|      显示 Dialog       |      C      |    Y     |    N    |
|     开启 Activity      |      C      |    Y     |    C    |
|     LayoutInflater     |      C      |    Y     |    C    |
|      开启 Service      |      Y      |    Y     |    Y    |
|     发送 Broadcast     |      Y      |    Y     |    Y    |
| 注册 BroadcastReceiver |      Y      |    Y     |    Y    |
|     加载 Resource      |      Y      |    Y     |    Y    |

如上这些 Context 提供的功能，在它的三个子类里面的支持情况不一，下面对`C(条件)`的情况予以说明：

### Application 显示 Dialog

使用 Application 可以显示 Dialog ，只是一般情况下不允许。一般的 Dialog 是应用级别的，这种 Dialog 必须依托于 Activity 才能显示，因为它需要一个 Window 来放置自己，而只有 Activity 才有，这是应用级别的 Dialog 的要求。如果把 Dialog 设置为系统级别的，那么它就可以依靠系统提供的某些功能来展示自己。

如果要使用 Application 显示 Dialog ，可使用`Dialog:getWindow():setType(WindowManager.LayoutParams.TYPE_TOAST)`将 Dialog 的类型设置为 Toast ，这是这个 Dialog 就是基于系统弹出的。

或者直接将 Dilaog 的类型设置为系统级的，但是普通应用没有这个权限，只有系统应用才能使用系统级 Dialog 。

### 非 Activity 中开启新的 Activity

每一个 Activity 都有它所在的任务栈，所以在使用一个 Activity 来启动 Activity 的时候，默认的就会把新开启的 Activity 存放到当前 Activity 所在的任务栈的栈顶，那么在 Application 或者 Service 中开启 Activity 的时候，就会因为没有任务栈存放 Activity 导致报错，这是可以使用`Intent:addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)`的方式，告知 AMS 要开启一个新的任务栈存放这个 Activity 就可以了。

### 非 Activity 使用 LayoutInflater

因为 Activity 继承于 ContextTheme ，所以对于 Activity 来说它有自己独自的 Theme ，就是用户自定义的 Theme ，但是在 Application 和 Serviece 中使用的仍然是系统默认的 Theme ，所以一般不会使用它们。

Activity 作为展示应用界面的基本组件，负责几乎所有的 UI 操作，所以一般情况下应用中关乎到 UI 操作使用到的 Context 都是 Activity 。

## Context 使用

因为 Context 用户较多的功能，并且每一个 Activity 和 Service 都对应着一个 Context ，所以如果在使用 Context 的时候使用不当导致 Context 无法正常回收很容易就会导致内存泄漏，关于内存泄漏说的最多久的就是 Activity 的引用问题，伴随着页面的打开和关闭，就会有 Activity 的创建和销毁，但是因为 Activity 中一般都会有比较复杂的业务逻辑，同时很多功能都需要使用 Context ，所以很多时候就会把 Activity 当作一个参数传递给其他类参与工作，如果 Activity 变成了另一个类的成员变量，但是在 Activity 生命周期结束的时候另一个类还没有被回收，就会导致还有对象持有 Activity 的引用导致 Activity 无法回收，所以在使用 Context 作为参数的时候一定要分外小心，一般有以下几种方法：

- 尽量使用 Application 作为参数使用
- 在必须要使用 Activity 的时候使用弱引用
- 确保持有 Activity 引用的对象的生命周期比 Activity 短
- 在 Activity 销毁的时候释放其他对象对 Activity 的引用
- 在 Activity 使用静态内部类（非静态内部类会隐式持有外部类也就是 Activity 实例）

## Context 获取

### `getApplicationContext()` & `getApplication()`

这两个方法得到的是一个对象，返回类型不同。前者是 Context 提供的方法，后者是 Activity 提供的方法，前者强调获得 Application 对应的 Context 对象，后者强调得到应用的 Application ，虽然它们确实是同一个对象。

### `Activity.this`

在 Activity 内部使用，获得当前 Activity 实例。

### `View:getContext()`

获得 View 所在 Context 实例，View 所需要的 Theme 、 Resource 等都是通过这个 Context 获得，一般是 View 所在的 Activity 实例。