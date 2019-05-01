---
title: Android 子线程更新 UI
date: 2019-03-28 22:00
tags:
	- android
---

在 Android 开发中有一个常识，那就是子线程中不能更新 UI ，所以，在使用子线程执行一个任务之后，总是还需要一个 Handler 切换到主线程，然后才能将结果展示在 View 中，否则就会抛出 CalledFromWrongThreadException 异常。

那么是否一定不能在子线程中更新 UI ？这种一种规则是怎么实现的？为什么要规定只能在主线程更新 UI ？

首先，要判断是否一定不能在子线程更新 UI ，需要先知道系统是怎么判断的。以 View 的`requestLayout()`为例，看看会调用哪些方法：

```java
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        // Only trigger request-during-layout logic if this is the view requesting it,
        // not the views in its parent hierarchy
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}

public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

可以看到，在 View 的`requestLayout()`方法中调用了 ViewParent 的`requestLayout()`方法，ViewParent 是一个接口，其实现类有两个，ViewGroup 和 ViewRootImpl ，而在 ViewGroup 中的实现与 View 一样，还会调用 ViewParent 的这个方法，所以最终，会调用 ViewRootImpl 的`requestLayout()`方法，其实现如上，其中`checkThread()`方法，人如其名，就是检测线程的方法，实现如下：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

所有的线程检测，都是通过这个方法完成的，那么如何才能在子线程更新 View 呢？只需要 View 的 ViewParent 也就是 ViewRootImpl 为空即可，这样就可以避免调用 ViewRootImpl 的`checkThread()`方法了，根据 [Android View 绘制流程](../android-view-draw-process) 可以知道，ViewRootImpl 是在`onResume()`阶段实例化并与 DecorView 产生关联的，所以只要是在 ViewRootImpl 实例化之前，确实是可以在子线程中更新 UI 的，但是在这种时候更新的 UI 没有 ViewRootImpl 的支持，并不会被绘制出来，用户也就无法看见。

那么为什么 Android 要规定只能在主线程更新 UI ？因为它的 UI 控件并不是线程安全的，不论是系统自带的一些 View ，还是开发者自定义 View ，也确实都没有考虑过多线程的问题。不过我觉得这并不是主要原因，或者应该反过来，正因为 Android 中只能在主线程更新 UI ，所以所有的 UI 控件都不需要考虑多线程的问题了。

那么如果没有这样规定会怎么样？就需要每个 UI 控件都能够处理多线程操作的问题，比如线程锁，使用线程锁会造成什么问题？一方面，UI 控件本身包含了很多的功能，如果再加上线程控制，无疑会让 UI 控件的复杂度再上一个台阶，另一方面，使用线程锁也会影响代码的执行效率，而 UI 的刷新频率决定着在用户看来应用是否卡顿，锁机制无疑是影响应用流畅性的一大因素。所以 Android 使用了另一种方式，规定所有的 UI 操作只能运行在主线程，也就是说 UI 操作始终是一个单线程的操作，在某些异步的情况下使用 Handler 切换到主线程这一步骤看似繁琐，但是换来了 UI 操作环境的纯净。