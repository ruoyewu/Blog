---
title: Android CoordinatorLayout Behavior 源码分析
date: 2018-2-28 13:00
tags:
	- android
	- android 源码
	- 工具
---

Android 官方的 Material Design 库让我们的 App 很轻松的就能实现一个比较炫酷的效果，如 CoordinatorLayout ，能够通过 Behavior 来协调它的各个子 View ，使得在 CoordiantorLayout 中的一个子 View 的状态发生变化的时候，对应的子 View 也能发生相应的变化。

现在就来具体了解一下 CoordinatorLayout 以及它的通常用法。

## CoordinatorLayout

>   CoordinatorLayout is a super-powered FrameLayout.

CoordinatorLayout 一个主要的使用方式就是与 CoordinatorLayout.Behavior 结合起来实现多个子 View 的联动，比如当 CoordinatorLayout 的一个子 View 移动的时候，另一个 View 也能随着它的移动而移动，或者其他的几个 View 都能跟随它的移动而移动，从而实现一个操作多种动态效果的结果。

## Behavior

Behavior 是 CoordinatorLayout 用来协调子 View 工作的最主要的媒介，使用 Bhhavior 的方法就是，给 CoordinatorLayout 的直接子 View 设置 Behavior，设置 Behavior 的方法有两种。**Behavior 只能作用于 CoordinatorLayout 的直接子 View。**第一种是在布局 xml 文件中，直接给 CoordinatorLayout 的直接子 View 加上`app:layout_behavior="对应的 Behavior 的路径"`的方式，指出当前 View 使用的 Behavior ，这种方法会使用到反射，所以不能将对应的 Behavior 混淆处理，同时使用者用方法的时候，对应的 Bhhavior 必须有 `CustomBehavior(Context, AttributeSet)`方法，这个方法就是使用 xml 加载类的时候需要调用的。

```java
package com.test.ui.behavior;

public class CustomBehavior extends CoordinatorLayout.Behavior<View> {
    public CustomBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    
    //下面写一些其他的监听操作
}
```

```xml
<CoordinatorLayout
                   android:layout_height="match_parent"
                   android:layout_width="match_parent">
    <View
          app:layout_behavior="com.test.ui.behavior.CustomBehavior"
          android:layout_height="match_parent"
          android:layout_width="match_parent"/>
</CoordinatorLayout>
```



另一种设置 Behavior 的方法，就是在代码里面直接设置，这里要用到`CoordinatorLayout.LayoutParams`这个类，通过直接调用对应的 Behavior 类的方式为 View 设置 Behavior 并保存在它的 LayoutParams 中，使用方法为：

```java

package com.test.ui.behavior;

public class CustomBehavior extends CoordinatorLayout.Behavior<View> {
    
    //下面写一些其他的监听操作
}
```

```java
View child = coordinatorLayout.getChildAt(0);
CoordinatorLayout.LayoutParams layoutParams = new CoordinatorLayout.LayoutParams(child.getLayoutParams());
layoutParams.setBehavior(new CustomBehavior());
chile.setLayoutParams(layoutParams);
```

在给一个 View 设置好 Behavior 之前，最主要的还是要确定一个这个 View 需要实现的功能，以及如何使用 Behavior 实现这个功能。先看一下 Behavior 提供的可以操作的方法：

```java
public static abstract class Behavior(V extends View) {
    // 构造函数，用于从代码中加载
    public Behavior() {}
    
    // 构造函数，用于从布局文件 xml 中加载
    public Behavior(Context context, AttributeSet attrs) {}
    
    // 调用 LayoutParams.setBehavior() 方法的时候调用此方法，或者加载到 xml 中的 Behavior 的时候调用
    public void onAttachedToLayoutParams(CoordinatorLayout.LayoutParams params){}
    
    // 为一个 View 设置了新的 Behavior 从而取代旧的 Behavior 的时候调用旧的 Behavior 的此方法
    public void onDetachedFromParams() {}
    
    // CoordinatorLayout 的 onInterceptTouchEvent 方法中逐个调用每个子 view 的 Behavior 的 此方法
    public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        return false;
    }
    
    // CoordinatorLayout 的 onTouchEvent 方法中调用
    public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        return false;
    }
    
    // 设置 View scrim 颜色
    @ColorInt
    public int getScrimColor(CoordinatorLayout parent, V child) {
        return Color.BLACK;
    }
    
    // 设置 View scrim 透明度
    @FloatRange(from = 0, to = 1)
    public void getScrimOpacity(CoordinatorLayout parent, V child) {
        return 0.f;
    }
    
    public boolean blocksInteractionBelow(CoordinatorLayout parent, V child) {
        return getScrimOpacity(parent, child) > 0.f;
    }
    
    // 判断 View child 是否 依赖于 dependency，在所有的布局出现变化的地方会调用，通常与 onDependentViewChanged 一起使用，先判断是否依赖，如果为真，则接着调用 onDependentViewChanged
    public boolean layoutDenpendsOn(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }
    
    // 与 layoutDenpendsOn 调用的地方差不多，同时 CoordinatorLayout 还提供了一个方法 dispatchDependentViewsChanged(View view) 方法，用于主动调用使用 view 作为依赖的 Behavior 调用这个方法。返回 true 表示 child 在此次方法调用中大小、位置等出现了变化
    public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }
    
    // 当 CoordinatorLayout 的一个子 View 别移除的时候，会遍历所有的以这个 View 为依赖的 Behavior，并调用此方法
    public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {}
    
    // CoordinatorLayout 测量 child 大小的时候会调用此方法，返回 ture 表示此方法对 child 作了大小的测算，不需要 CoordinatorLayout 再作测算
    public boolean onMeasureChild(CoordinatorLayout parent, V child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
        return false;
    }
    
    // 与 onMeasureChild 类似，用于替代 Coordinatorlayout 默认的 onLayoutChild 方法
    public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection) {
        return false;
    }
    
    public static void setTag(View child, Object tag) {
        final LayoutParams lp = child.getlayoutParams();
        lp.mBehaviorTag = tag;
    }
    
    public static Object getTag(View child) {
        final LayoutParams lp = child.getLayoutParams();
        return lp.mBehaviorTag;
    }
    
    // 在一个嵌套滑动开始的时候调用，用于检测是否当前 Behavior 需要检测整个滑动过程，只有在这个方法中返回 true 的 Behavior，才会接收到下面一系列的关于 NestedScroll 的方法的调用。
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, V child, View directTargetChild, View target, int axes, int type) {
        return false;
    }
    
    // 在 onStartNestedScroll 中返回 true 的 Behavior 都会紧接着调用这个方法
    public void onNestedScrollAccepted(CoordinatorLayout coordinatorLayout, V child, View directTargetChild, View target, int axes, int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            onNestedScrollAccepted(coordinator, child, directTargetChild, target, axes);
        }
    }
    
    // 滑动停止时调用
    public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View directTargetChild, View target, int axes, int type) {
        
    }
    
    // 滑动过程中调用
    public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type）{}
    
    // 监测到手势操作，View 即将滑动的时候调用
    public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target, int dx, int dy, int[] consumed, int type) {}
    
    // 快速拉动之后的惯性滑动，返回 true 表示对这个状态进行了响应
    public boolean onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target, float velocityX, float velocityY, boolean consumed) {
        return false;
    }
    
    // 一次快速滑动的开始调用，返回 true 表示 CoordinatorLaout 需要消费这个事件
    public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target, float velocityX, float velocityY) {
        return false;
    }
    
    public WindowInsetsCompat onApplyWindowInsets(CoordinatorLayout coordinatorLayout, V child, WindowInsetsCompat insets) {
        return insets;
    }
    
    public boolean onRequestChildRectangleOnScreen(CoordinatorLayout coordinatorLayout, V child, Rect rectangle, boolean immediate) {
        return false;
    }
    
    public void onRestoreInstanceState(CoordinatorLayout parent, V child, Parcelable state) {}
    
    public Parcelable onSaveInstanceState(CoordinatorLayout parent, V child) {
        return BaseSaveState.EMPTY_STATE;
    }
    
    public boolean getInsetDodgeRect(CoordinatorLayout parent, V child, Rect rect) {
        return false;
    }
}
```

Behavior 提供了大量的方法使得 CoordinatorLayout 的子 View 之间的协调能力大大增强，可以很方便地根据 CoordinatorLayout 的状态、CoordinatorLayout 其他子 View 的状态进行适应性调整。如上所示，就是 Behavior 提供的基本的接口，可以看到，这些方法主要分为三种功能：

1.  使一个 View 监听另一个 View 的状态，当被监听者状态改变的时候，通知监听者作相应的改变。
2.  使一个 View 监听 CoordinatorLayout 的各种滑动，并对不同的滑动作出响应。
3.  使一个 View 在 CoordinatorLayout 某些状态发生改变的时候，通知 View 作出改变，如最后的几个方法。

使用 Behavior 最根本的思想就是，Behavior 应该与 View 是一一对应的，Behavior 作为 View 与 CoordinatorLayout 之间的媒介，将 CoordinatorLayout 的信息传递给 View 并通过这些方法的返回值给予 CoordinatorLayout 一定的反馈，使得处于 CoordinatorLayout 之下的 View 能够完成一些复杂的联动效果，一句话就是，让界面更加炫酷，更加流畅。

## 参考

[https://appkfz.com/2015/11/12/mastering-coordinator/](https://appkfz.com/2015/11/12/mastering-coordinator/)

