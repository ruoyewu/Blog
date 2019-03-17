---
title: Android 属性动画
date: 2018-04-19 10:40
tags:
	- android
---

## Android 动画

界面时用户与程序交互的媒介，一个好的界面会让用户更愿意使用这个程序，动画则能很大程度上提升用户的操作体验，使得界面的切换不那么突兀，给人一种水到渠成的感觉，所以在合适的时间、合适的地点使用动画非常必要，Android 提供了几种不同类型的实现动画的方式，视图动画、属性动画和帧动画。

## 属性动画

属性动画的基本实现方法就是，通过一系列较小幅度的对一个 View 的各种属性（如位置、大小、透明度）的更改，实现用户眼中“动”的效果，而且这个动画是真实的动，区别于视图动画。与属性动画相关的类是`android.animation.ObjectAnimator`，它继承于`android.animation.ValumeAnimator`。

## ObjectAnimator

使用 ObjectAnimator 的方法有主要分两种，一种是通过传入 Property 和与这个 Property 对应的 target 完成，如`OjbectAnimator.ofFloat(T target, Property<T, Float> property, float... values)`，另一种是通过传入要更改值的属性名，通过反射机制完成对应属性的值的更改，如`ObjectAnimator.ofFloat(Object target, String propertyName, float... values)`，ObjectAnimator 执行工作的主要方法就是从 ValueAnimator 继承来的`animateValue(float fraction)`，在这里会调用 PropertyValuesHolder 的`setAnimatedValue(Object target)`方法完成对 target 的属性的赋值从而改变 target 的某个状态来实现动画效果。

## PropertyValuesHolder

PropertyValuesHolder 的构造方法分为两种，一种是传入 Property ，另一种则是传入 propertyName ，与 ObjectAnimator 的构造方法相对应，并且在准备启动动画的时候对其作初始化`setupSetterAndGetter(Object target)`，主要工作就是初始化关键帧，如果需要的话，使用反射得到对应属性的 getter 和 setter 方法，用于后续。然后在动画执行的过程中，ValueAnimator 会不断调用它的`animateValue(float fraction)`方法更新中间值，ObjectAnimator 会在这个方法中调用`setAnimatedValue(Object target)`方法更新 target 的值。

`android.animation.ProperValuesHolder:setAnimatedValue(Object)`

```java
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```

看这个方法的实现就可以知道，分别通过`Property:set()`和`Method:invoke()`方法给对应的属性赋值，即可。

上述流程都是基于已经实现的 ValueAnimator 来讲述的，还要了解 ValueAnimator 的实现过程才能了解属性动画真正的过程，比如是什么机制能够使得 ValueAnimator 能够不停调用`animateValue(float fraction)`方法，又是什么通过什么方法算出两个关键帧中间的值的，这些都要从 ValueAnimator 看起。

## ValueAnimator

ValueAnimator 的工作是，我们给定至少两个关键帧（初始值和最终值），它就可以根据设置的插值器得到一系列的中间值，ObjectAnimator 就是以 ValueAnimator 为基础而成。当调用了它的`start()`方法的时候，就会准备开始动画的执行，在这个过程中首先会调用`addAnimationCallback(0)`，这个方法负责将 ValueAnimator 与 AnimationHandler 联系起来，而 ValueAnimator 工作的过程，就是依靠 AnimationHandler 一帧一帧地调用`ValueAnimator:doAnimationFrame(long)`方法来告知 ValueAnimator 进行数值更新的，然后再调用`ValueAnimator:startAnimation()`进行 ValueAnimator 自身的初始化，如内部的 PropertyValuesHolder 的初始化以及调用`AnimatorListener:onAnimationStart()`。然后这些准备工作做完之后，就可以等待 AnimationHandler 发起刷新调用就好了，进入`doAnimationFrame(long frameTime)`方法，在这里进行了动画的延迟判断、状态判断等，并最终进入`ValueAnimator:animateBasedOnTime(long currentTime)`，根据 currentTime 与 startTime 的比较，得到 fraction ，然后根据是否应该重复动画得到 currentIterationFraction ，使得 currentIterationFraction 处于 [0，1] 之间的值，然后调用`ValueAnimator:animateValue(float fraction)`，在这个方法中，PropertyValuesHolder 会根据这个 fraction 调用`PropertyValuesHolder:calculateValue(float fraction)`得到最终的中间值，而这个值是根据给定义 ValueAnimator 的时候给定的一系列关键帧计算得到的。中间值计算完成之后，就会调用`AnimatorUpdateListener:onAnimationUpdate()`方法将这个结果分发出去。

具体的通过关键帧和时间计算出中间值的过程，交由`android.animation.Keyframes`完成。

## KeyFrames

Keyframes 是一个接口类，需要实现其中的方法完成工作，如`android.animation.KeyframeSet`，它计算中间值的方法是，通过当前所处的时间，得到当前所处的关键帧的范围，然后再根据两个关键帧的时间，通过插值的方法，取当前时间所对应的值，就得到结果了。

如有关键帧`{0, 1}, {0.5, 5}, {1, 3}`，那么对于时间 0.7 来说，首先可以知道它处于`{0.5, 5}`-`{1, 3}`阶段，然后通过计算`5 + (3 - 5) * (0.7 - 0.5) / (1 - 0.5) = 4.2 `，这个 4.2 就是通过插值计算出来的中间值。

## AnimationHandler

ValueAnimator 的工作条件就是 AnimationHandler 提供的，它们之间通过设置 Callback 和 Callback 的回调完成通信。AnimationHandler 是一个单例，也就是说所有的 ValueAniamtor 都是同用的这一个实例，那么 AnimationHandler 什么时候开始工作、什么时候停止工作就是一个需要考虑的问题。

`AnimationHandler:addAnimationFrameCallback(AnimationFrameCallback callback, long delay)`

```java
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        getProvider().postFrameCallback(mFrameCallback);
    }
    if (!mAnimationCallbacks.contains(callback)) {
        mAnimationCallbacks.add(callback);
    }
    if (delay > 0) {
        mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
    }
}
```

在 ValueAnimator 的准备运行阶段会调用这个方法添加 Callback ，可以看到，当 AnimationHandler 的 Callback 列表数量由 0 变 1 的时候，就会调用一次`AnimationFrameCallbackProvider:postFrameCallback(Choreographer.FrameCallback callback)`方法，然后这个方法是对`android.view.Choreographer`的一个包装，所有对 AnimationFrameCallbackProvider 的调用最终会调用 Choreographer ，而 Choreographer 的工作就是在很短的时间（可以看作一帧）之后发起回调，具体的回调代码：

```java
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            getProvider().postFrameCallback(this);
        }
    }
};
```

这里的`doFrame(long frameTimenanos)`就是 Choreographer 在一帧时间后发起的回调方法，在这里 AnimationHandler 首先调用了`AnimationHandler:doAnimationFrame(long frameTime)`将这一帧分发给它所有的 Callback ，告知它们到了刷新的时间了，然后如果现在还存在 Callback ，就会再次调用`postFrameCallback()`方法向 Choreographer 请求下一帧的回调，以此构成循环。

## Choreographer

AnimationHandler 通过 Choreographer 获得一系列以单位帧时间为间隔的序列，然后将其反馈给 ValueAnimator ，同时 AnimationHandler 与 Choreographer 之间也是通过 Callback 进行沟通。Choreographer 是一个线程内的单例，每次通过`Choreographer:postFrameCallback(FrameCallback callback)`调用的时候，Choreographer 会短暂地延时一段很小的时间后调用 callback ，其内部使用了锁机制完成线程同步。Choreographer 为动画、输入系统和绘制系统提供一个很短暂的时间间隔，动画会在这个间隔内完成控件的重绘等，从而利用视觉暂留效果使用户看到了“动的画面”，它就类似于 Android UI 系统的一个时钟信号。Choreographer 内部通过 Handler 机制或者是 JNI 调用 C/C++ 层次的代码完成时间间隔后的唤醒。

## Property

Property 是一个抽象类，要使用这个类的时候需要先继承这个类，可以直接先看一个 Property 的实例。

`android.view.View.ALPHA`

```java
public static final Property<View, Float> ALPHA = new FloatProperty<View>("alpha") {
    @Override
    public void setValue(View object, float value) {
        object.setAlpha(value);
    }
    @Override
    public Float get(View object) {
        return object.getAlpha();
    }
};
```

这就是一个用于设置 View 的透明度的 Property 实例 ALPHA ，使用还是挺方便的，View 还提供了用于设置偏移量等对应的 Property ，需要的时候直接使用即可。

Android 的属性动画的原理就是这样，OjbectAniamtor 通过反射或者回调给对应的修改对象的变量值，再通过 ValueAnimator 得到每一帧的变量的值，ValueAnimator 通过 AnimationHandler 完成每过一段时间的回调，然后在自己内部完成了通过时间计算变量的中间值的过程，AnimationHandler 调用 Choreographer 方法使得在一个短暂的时间间隔时候回调自己的方法，然后将这个回调分发给所有的 ValueAnimator ，然后再次调用 Choreographer 的方法等待下次时间间隔的到来，Choreographer 则是负责生成时间间隔，它通过调用 Handler 或者通过调用本地 C/C++ 方法完成这项功能。这样一系列功能的调用，就是我们看到了各种各样的动画。