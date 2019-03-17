---
title: Android View 绘制流程
date: 2019-03-17 10:24
tags:
	- android
---

前面讲到 [Android View 加载流程](../android-view-inflate-process)，使用 LayoutInflater 将 xml 文件转变成 View ，但是还需要将 View 绘制出来，才能被用户看到，这一过程为绘制流程。由于 Android 的整个 View 是以树的形式出现的，所以很多关于 View 的机制，如加载、事件分发等，都是以递归的方法完成的，绘制流程也不意外。

### Resume

对于每一个 view 而言，绘制分为三个阶段：测量、布局、绘制。只有在 view 经历过绘制之后，才能被用户看到。对应着 Activity 的生命周期，在 resume 阶段，activity 变为前台用户可以看到的界面，简单推测 view 的绘制应该也就是在这个阶段，那么就从`ActivityThread.handleResumeActivity()`方法出发，看看这之后的流程是什么。

#### ActivityThread

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    // TODO Push resumeArgs into the activity for consideration
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    // ...
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        // ...
    }
    if (r.window == null && !a.mFinished && willBeVisible) {
        View decor = r.window.getDecorView();
        // ..
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            } else {
                // The activity will get a callback for this {@link LayoutParams} change
                // earlier. However, at that time the decor will not be set (this is set
                // in this method), so no action will be taken. This call ensures the
                // callback occurs with the decor set.
                a.onWindowAttributesChanged(l);
            }
        }
        // If the window has already been added, but during resume
        // we started another activity, then don't yet make the
        // window visible.
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }
    // ...
}
```

DecorView 是在`onCreate()`是通过`setContentView()`的过程中创建的，通过调用`WindowManager.addView()`方法将 DecorView 添加到对窗口中，然后才能将其展示出来。

#### WindowManager

对应方法代码如下：

```java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}

public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // ...
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    // ...
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        // ...
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

WindowManager 的实现类是 WindowManagerImpl ，可以看到，在这个方法中创建了一个 ViewRootImpl 对象，ViewRootImpl 本身并不是一个 View ，但是它作为视图层次中最顶级的存在，实现了 ViewParent 接口，有很多关于 View 的一些操作，在 View 和 WindowManager 中间承担着中间人的角色。比如 View 的绘制流程，就是由这个类总掌的。

#### ViewRootImpl

上面的代码中，最后一句是`root.setView()`，此时便将 DecorView 这个顶级 View 作为参数传递给了 ViewRootImpl ，在之后的绘制流程中，ViewRootImpl 就是通过这个 DecorView 向整个 ViewTree 发布命令的。

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            // ...
            /* = WindowManagerImpl.ADD_OKAY; */
            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();
            // ...
        }
    }
}
```

在`setView()`方法中与 View 绘制最相关的大概就是`requestLayout()`这一句了。字面意思为「请求布局」，具体的逻辑在下面。

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

上面的代码主要做了两件事：

1.  发送一个阻塞同步消息的栅栏
2.  给 Choreographer 发送一个回调 Runnable

第一步发送一个用于阻塞 Handler 消息的栅栏，使得在这之后添加的同步消息都不能立刻执行，直到将这个栅栏移出为止。然后下一步就向 mChoreographer 提交了一个 Runnable ，经过一系列的调用之后，这个 Runnable 会被执行。这个 Runnable 是什么？

```java
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

也就是说，后续需要调用`doTraversal()`方法。

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
        performTraversals();
        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

也是两个步骤：

1.  移除刚刚添加的栅栏
2.  调用`performTraversals()`

第一步对应上上一步添加的同步消息栅栏，使得同步消息可以继续执行，第二步就调用了`performTraversals()`方法，这个方法总掌 View 的整个绘制流程。方法中的代码很多，其中有三个方法完成了 view 的绘制过程，分别是`performMeasure()` `performLayout()` 和 `performDraw()`，对应着测量、布局和绘制。

### Measure

调用了`performMeasure()`方法之后就正式进入了 Measure 的阶段。

#### MeasureSpec

测量阶段，就是测量一个 View 到底有多长多宽，下面才好给它分配对应的空间，供其展示自己，在开发的过程中，每在 xml 布局文件中增加一个 View 的时候，编辑器就会自动给 View 加上对应的设置宽和高的代码，如下：

```xml
<View
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
```

所有的 View 都不能缺少这两个属性，否则就会报错。那么既然所有的 View 都需要设置宽和高，进行测量的意义何在？问题就在于上面的两种设置，它们并没有给定一个确定的数值，而是设置了一种模式。第一个`match_parent`，意为父 View 能给自己分配多少，自己就要多少，第二个`wrap_content`，只要满足能够展示自己的内容或者满足自己的子 View 的需要的长宽就够了。Android 为了满足这两种要求，必须要有一个根据 View 的要求计算出正确的数值的功能。

所以对于任何一个 View 的长宽来说，都需要两个参数，测量模式和测量数值。测量模式有三种：

1.  UNSPECIFIED，父 View 没有对当前 View 的限制，想要多大都可以
2.  EXACTLY，父 View 给当前 View 一个确定的数值，子 View 没有选择权
3.  AT_MOST，父 View 给出了一个上限，子 View 可自由选择，但最多不超过上限

同时还有父 View 提供的数值，这两个数据是伴随着测量的整个过程的，使用上自然十分频繁，为了提高传输效率，可以将这两个数据合成一个数据，MeasureSpec 完成的就是这样的功能，在合成之后的数据中，模式占整形的高 2 位，低 30 位则是数值，合成与分解的方法都由 MeasureSpec 提供。

#### `performMeasure()`

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

第 7 行中调用了 mView 的`measure()`方法，这里的 mView 就是 DecorView 实例，它继承于 FrameLayout ，但是 FrameLayout 以及 ViewGroup 都没有对`measure()`方法的重写，此时要看 View 类中关于这一方法的实现：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int oWidth  = insets.left + insets.right;
        int oHeight = insets.top  + insets.bottom;
        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
    }
    // Suppress sign extension for the low bytes
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
    // Optimize layout by avoiding an extra EXACTLY pass when the view is
    // already measured as the correct size. In API 23 and below, this
    // extra pass is required to make LinearLayout re-distribute weight.
    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
            || heightMeasureSpec != mOldHeightMeasureSpec;
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
            && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
            && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
    final boolean needsLayout = specChanged
            && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
    if (forceLayout || needsLayout) {
        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
        resolveRtlPropertiesIfNeeded();
        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                    + getClass().getName() + "#onMeasure() did not set the"
                    + " measured dimension by calling"
                    + " setMeasuredDimension()");
        }
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;
    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```

这是一个 final 方法，所以它的所有子类都没有对这一方法的重写，一般子类如果需要就测量方面的问题做一些自定义操作，都是通过重写`onMeasure()`方法完成的。那这个方法做了什么事？

上面那段代码主要判断了一件事：是否需要 layout ，以及使用何种方式 layout 。通过两个 boolean 值 forceLayout 和 needsLayout ，确定了最终的方式，如果 forceLayout 为真或者第一次以当前的这种宽和高出现，则需要调用`onMeasure()`方法。

上面说到，如果要自定义一个 ViewGroup 的话，那么这个 ViewGroup 的`onMeasure()`方法就需要完成测量子 View 的任务，所以`onMeasure()`在普通的 View 和 ViewGroup 中有不同的实现。

#### 在 View 中

在 View 类中有`onMeasure()`的实现：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}

protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;
        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

经过一连串的调用，mMeasureWidth 得到值，测量结束。如果 View 还有额外的需求，如 TextView 需要空间展示文字的时候，也可以重写`onMeasure()`方法：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    if (widthMode == MeasureSpec.EXACTLY) {
        // Parent has told us how big to be. So be it.
        width = widthSize;
    } else {
        // 计算展示文字所需空间
    }
    // ...
    if (heightMode == MeasureSpec.EXACTLY) {
        // Parent has told us how big to be. So be it.
        height = heightSize;
        mDesiredHeightAtMeasure = -1;
    } else {
        int desired = getDesiredHeight();
        height = desired;
        mDesiredHeightAtMeasure = desired;
        if (heightMode == MeasureSpec.AT_MOST) {
            height = Math.min(desired, heightSize);
        }
    }
    // ...
    setMeasuredDimension(width, height);
}
```

如上，结合文字所需空间进行对 View 的测量之后再调用`setMeasureDimension()`方法设置最终的大小。

#### 在 ViewGroup 中

ViewGroup 是一个抽象类，其`onMeasure()`就是一个抽象方法，以 LinearLayout 为例：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

首先，根据 LinearLayout 的方向，再具体选择一个方法，如`measureVertical()`：

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    
    // See how tall everyone is. Also remember max width.
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        
        // ...
        final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
        if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
            // Optimization: don't bother measuring children who are only
            // laid out using excess space. These views will get measured
            // later if we have space to distribute.
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            skippedMeasure = true;
        } else {
            if (useExcessSpace) {
                // The heightMode is either UNSPECIFIED or AT_MOST, and
                // this child is only laid out using excess space. Measure
                // using WRAP_CONTENT so that we can find out the view's
                // optimal height. We'll restore the original height of 0
                // after measurement.
                lp.height = LayoutParams.WRAP_CONTENT;
            }
            // Determine how big this child would like to be. If this or
            // previous children have given a weight, then we allow it to
            // use all available space (and we will shrink things later
            // if needed).
            final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
            // 第一步
            measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                    heightMeasureSpec, usedHeight);
            // ...
        }
        // ...
        i += getChildrenSkipCount(child, i);
    }
    // ...
    if (useLargestChild &&
            (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
        mTotalLength = 0;
        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            
            // ...
            final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                    child.getLayoutParams();
            // Account for negative margins
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }
    }
    
    // ...
    if (skippedMeasure
            || ((sRemeasureWeightedChildren || remainingExcess != 0) && totalWeight > 0.0f)) {
        float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;
        mTotalLength = 0;
        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            
            // ...
            if (childWeight > 0) {
                
                // 第二步
                final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.max(0, childHeight), MeasureSpec.EXACTLY);
                final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                        lp.width);
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
                // Child may now not fit in vertical dimension.
                childState = combineMeasuredStates(childState, child.getMeasuredState()
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
            }
            
            // ...
        }
        // Add in our padding
        mTotalLength += mPaddingTop + mPaddingBottom;
        // TODO: Should we recompute the heightSpec based on the new total length?
    } else {
        alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                       weightedMaxWidth);
        // We have no limit, so make all weighted views as tall as the largest child.
        // Children will have already been measured once.
        if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
            for (int i = 0; i < count; i++) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                float childExtra = lp.weight;
                if (childExtra > 0) {
                    child.measure(
                            MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                    MeasureSpec.EXACTLY),
                            MeasureSpec.makeMeasureSpec(largestChildHeight,
                                    MeasureSpec.EXACTLY));
                }
            }
        }
    }
    
    // ...
    // 第三步
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);
    if (matchWidth) {
        forceUniformWidth(count, heightMeasureSpec);
    }
}
```

方法比较长，整个流程大致分为三步：

1.  测量所有需要测量的子 View
2.  将 LinearLayout 剩余的空间按照 weight 的比例将其分配给其子 View
3.  计算出所有子 View 的大小之后，根据子 View 所需的空间和父 View 提供的空间和测量模式，设置 LinearLayout 的大小

其中，如果一个子 View 同时设置了 height 和 weight 都不为 0 ，那么它在第一、第二步都会被涉及到，并且两次分配的高度是相加的，weight 这个设置只有在 LinearLayout 的测量模式为 EXACTLY 时才有用，因为需要使用到它的剩余空间。

测量子 View 的方法主要是`measureChildBeforeLayout()`，相关的方法如下：

```java
void measureChildBeforeLayout(View child, int childIndex,
        int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
        int totalHeight) {
    measureChildWithMargins(child, widthMeasureSpec, totalWidth,
            heightMeasureSpec, totalHeight);
}

protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

这一连串调用最终会调用到`child.measure()`方法，进入下一个循环。在这之前，首要的工作是确定确定两个参数：withMeasureSpec 和 heightMeasureSpec ，即 child 的测量模式，以及它能够给子 View 提供的大小。MeasureSpec 是根据父 View 的 MeasureSpec 和子 View 的 LayoutParams 共同确定的，具体的逻辑处理就在`getChildMeasureSpec()`方法中。它有三个参数：

1.  spec: Int，父 View 提供的 MeasureSpec
2.  padding: Int，父 view 的 padding 和子 View margin 之和，即留白所需的空间
3.  childDimension: Int，在 LayoutParams 中对子 View 宽高的设置

父 View 能够为子 View 提供的最大空间，是 spec 中的数值部分与 padding 之差。所以最终，`getChildMeasureSpec()`方法实际上拥有的是三个数据：

1.  specMode，父 View 模式
2.  size，父 View 提供的空间
3.  childDimension，子 View 所需的空间，-1 代表 match_parent，-2 代表 wrap_content ，大于等于 0 则是对应着具体的数值

之后便是 switch 和 if 之间的嵌套，switch 是对父 View 的测量模式的判断，if 是对子 View 的模式的判断，共 9 种情况，每一种都写在上面的代码中了。可由下面的表格概括（最左一列表示父 View 模式，最上一行表示子 View 模式，中间 3\*3 表示结果，childDimension 是子 View 定义的空间，size 为父 View 能够提供的空间）：

| 父 View 模式/子 View 模式 |          \>= 0          |   match_parent    |   wrap_content    |
| :-----------------------: | :---------------------: | :---------------: | :---------------: |
|        UNSPECIFIED        | EXACTLY, childDimension | UNSPECIFIED, size | UNSPECIFIED, size |
|          EXACTLY          | EXACTLY, childDimension |   EXACTLY, size   |   AT_MOST, size   |
|          AT_MOST          | EXACTLY, childDimension |   AT_MOST, size   |   AT_MOST, size   |

计算得出子 View 的 MeasureSpec 之后，下一步就可以根据这个数值继续调用`child.measure()`方法对其及其子 View 进行测量，步骤同上。

### Layout

在调用了`performMeasure()`方法之后，接着就会调用`performLayout()`方法，在这个方法中会调用 DecorView 的`layout()`方法，从而进入 ViewTree 的 layout 过程：

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;
    final View host = mView;
    if (host == null) {
        return;
    }
    try {
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        // ...
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

在 view 中与 layout 相关的也有两个方法，`layout()`和`onLayout()`。

#### `layout()`

下面是 View 中`layout()`方法的实现：

```java
public void layout(int l, int t, int r, int b) {
    // ...
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        
        // ...
    }
    // ...
}


```

在这个方法中，首先调用了`setFrame()`方法，将这个 View 在父 View 中的位置设置完成，包括四个参数：left right top bottom ，这些都是相对于父 View 而不是相对于屏幕的。然后接着调用`onLayout()`方法，在 View 中这是一个空方法，主要是为了给 ViewGroup 重写，以完成对子 View 的布局。

#### `onLayout()`

以 LinearLayout 为例，它的`onLayout()`方法为：

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```

不同的排列方式对应着不同的布局方法，以`layoutVertical()`为例：

```java
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;
    int childTop;
    int childLeft;
    // Where right end of child should go
    final int width = right - left;
    int childRight = width - mPaddingRight;
    // Space available for child
    int childSpace = width - paddingLeft - mPaddingRight;
    final int count = getVirtualChildCount();
    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
    switch (majorGravity) {
       case Gravity.BOTTOM:
           // mTotalLength contains the padding already
           childTop = mPaddingTop + bottom - top - mTotalLength;
           break;
           // mTotalLength contains the padding already
       case Gravity.CENTER_VERTICAL:
           childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
           break;
       case Gravity.TOP:
       default:
           childTop = mPaddingTop;
           break;
    }
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                            + lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;
                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }
            childTop += lp.topMargin;
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
            i += getChildrenSkipCount(child, i);
        }
    }
}

private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```

LinearLayout 的`layoutVertical()`方法的整个实现，整体来看还是比较简单的，首先，根据设置的 Vertical Gravity 决定从哪里开始布局，childTop 决定子 View 的纵向起点，然后再根据其 Horizontal Gravity 决定 childLeft ，横向起点，最后调用`setChildFrame()`方法，调用子 View 的`layout()`方法，此时 ViewGroup 的工作就完成了。总而言之，就是根据 gravity、padding、margin 等数据，决定一个 View 到底该出现在什么位置才是正确的。

### Draw

绘制流程的最后一步，Draw，由 ViewRootImpl 中的`performDraw()`开启，继而调用`draw()`、`drawSoftware()`方法，最后调用 DecorView 的`draw()`方法，进入 ViewTree 的 Draw 过程。

#### `draw()`

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */
    // Step 1, draw the background, if needed
    int saveCount;
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }
    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);
        // Step 4, draw the children
        dispatchDraw(canvas);
        drawAutofilledHighlight(canvas);
        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);
        if (debugDraw()) {
            debugDrawFocus(canvas);
        }
        // we're done...
        return;
    }
    /*
     * Here we do the full fledged routine...
     * (this is an uncommon case where speed matters less,
     * this is why we repeat some of the tests that have been
     * done above)
     */
    boolean drawTop = false;
    boolean drawBottom = false;
    boolean drawLeft = false;
    boolean drawRight = false;
    float topFadeStrength = 0.0f;
    float bottomFadeStrength = 0.0f;
    float leftFadeStrength = 0.0f;
    float rightFadeStrength = 0.0f;
    // Step 2, save the canvas' layers
    int paddingLeft = mPaddingLeft;
    final boolean offsetRequired = isPaddingOffsetRequired();
    if (offsetRequired) {
        paddingLeft += getLeftPaddingOffset();
    }
    int left = mScrollX + paddingLeft;
    int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
    int top = mScrollY + getFadeTop(offsetRequired);
    int bottom = top + getFadeHeight(offsetRequired);
    if (offsetRequired) {
        right += getRightPaddingOffset();
        bottom += getBottomPaddingOffset();
    }
    final ScrollabilityCache scrollabilityCache = mScrollCache;
    final float fadeHeight = scrollabilityCache.fadingEdgeLength;
    int length = (int) fadeHeight;
    // clip the fade length if top and bottom fades overlap
    // overlapping fades produce odd-looking artifacts
    if (verticalEdges && (top + length > bottom - length)) {
        length = (bottom - top) / 2;
    }
    // also clip horizontal fades if necessary
    if (horizontalEdges && (left + length > right - length)) {
        length = (right - left) / 2;
    }
    if (verticalEdges) {
        topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
        drawTop = topFadeStrength * fadeHeight > 1.0f;
        bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
        drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
    }
    if (horizontalEdges) {
        leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
        drawLeft = leftFadeStrength * fadeHeight > 1.0f;
        rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
        drawRight = rightFadeStrength * fadeHeight > 1.0f;
    }
    saveCount = canvas.getSaveCount();
    int solidColor = getSolidColor();
    if (solidColor == 0) {
        if (drawTop) {
            canvas.saveUnclippedLayer(left, top, right, top + length);
        }
        if (drawBottom) {
            canvas.saveUnclippedLayer(left, bottom - length, right, bottom);
        }
        if (drawLeft) {
            canvas.saveUnclippedLayer(left, top, left + length, bottom);
        }
        if (drawRight) {
            canvas.saveUnclippedLayer(right - length, top, right, bottom);
        }
    } else {
        scrollabilityCache.setFadeColor(solidColor);
    }
    // Step 3, draw the content
    if (!dirtyOpaque) onDraw(canvas);
    // Step 4, draw the children
    dispatchDraw(canvas);
    // Step 5, draw the fade effect and restore layers
    final Paint p = scrollabilityCache.paint;
    final Matrix matrix = scrollabilityCache.matrix;
    final Shader fade = scrollabilityCache.shader;
    if (drawTop) {
        matrix.setScale(1, fadeHeight * topFadeStrength);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, right, top + length, p);
    }
    if (drawBottom) {
        matrix.setScale(1, fadeHeight * bottomFadeStrength);
        matrix.postRotate(180);
        matrix.postTranslate(left, bottom);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, bottom - length, right, bottom, p);
    }
    if (drawLeft) {
        matrix.setScale(1, fadeHeight * leftFadeStrength);
        matrix.postRotate(-90);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, left + length, bottom, p);
    }
    if (drawRight) {
        matrix.setScale(1, fadeHeight * rightFadeStrength);
        matrix.postRotate(90);
        matrix.postTranslate(right, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(right - length, top, right, bottom, p);
    }
    canvas.restoreToCount(saveCount);
    drawAutofilledHighlight(canvas);
    // Overlay is part of the content and draws beneath Foreground
    if (mOverlay != null && !mOverlay.isEmpty()) {
        mOverlay.getOverlayView().dispatchDraw(canvas);
    }
    // Step 6, draw decorations (foreground, scrollbars)
    onDrawForeground(canvas);
    if (debugDraw()) {
        debugDrawFocus(canvas);
    }
}
```

在这个方法中，规定了 Draw 过程的 7 个步骤：

1.  绘制背景
2.  保存 canvas 
3.  绘制内容
4.  绘制子 View
5.  绘制边缘并恢复 canvas
6.  绘制前景（如 scrollbar）
7.  绘制默认的焦点高亮

代码执行有两个分支，其中一个分支不需要执行第 2、5 步，另一个分支则没有第 7 步。主要的步骤就是 1、3、4、6这四个，其功能描述也很清晰，分别是背景、内容、子 View 和前景，它们也是一层一层覆盖的关系，所以在这里可以看到，一个 View 的前景不仅会覆盖它的内容，还会覆盖子 View 。

绘制内容调用的方法为`onDraw()`，所以在自定义 View 的时候，一般都是在这个方法中进行绘制。绘制子 View 调用了`dispatchDraw()`将这个 Draw 事件分发出去，这个方法在 View 中是一个空方法，ViewGroup 中对其进行了重写。

#### `dispatchDraw()`

这个方法通过`drawChild()`将 Draw 事件传递给子 View ，子 View 也会进入`draw()`方法从而继续上述步骤。在`dispatchDraw()`方法中出现的 child 有三种：

1.  transient child
2.  child
3.  disappearing child

只有第二个是 ViewGroup 的常驻 child ，transient child 只会在 Draw 的过程中展示，它不会占用父 View 的空间，也不会参与用户交互等，一般用在执行动画的过程中。disappearing child 是那些被移除或者隐藏的 children ，但是它们还需要隐藏动画，隐藏的过程是需要在 draw 过程中绘制出来的。

#### `onDraw()`

在这个过程中主要使用到的类有 Canvas 、Paint ，使用 Canvas 的各种`drawXxx()`方法，能够画出各种形状，然后再以 Paint 设置画笔的方式、颜色等等，所有的用户看到的界面几乎都是以这种方式展示出来的。

### 关于自定义

Android 默认提供了很多的控件，使用这些控件基本能够完成大部分场景页面的编辑，但是有些时候可能会需要一些特殊的效果，这时就需要自定义 View 来实现，自定义 View 一般分为两种情况，View or ViewGroup 。

#### 自定义 View

自定义 View 的时候一般只涉及到 measure 和 draw 过程，其中 measure 过程也多是针对 AT_MOST 模式下的情况，根据 View 需要展示的内容，确定其大小。

View 一般来说主要还是用来展示某个东西的，即内容的绘制，在 draw 过程，对应着`onDraw()`方法，比如需要一个能够展示五彩斑斓颜色的圆形的 View ，一般就是重写`onDraw()`方法，在 Canvas 中依照所需进行绘画。

#### 自定义 ViewGroup

虽然 ViewGroup 也是 View 的一个子类，但是它存在的意义变得不一样了，View 主要是用来展示内容，而 ViewGroup 更多的是用来展示 View 的，所以在一个 ViewGroup 里面更重要的是确定子 View 应该有多大，应该放在哪里，即对应的 measure 和 layout 过程，相对的关于 draw 的过程，ViewGroup 里面实现的`dispatchDraw()`方法已经足够了。

### 总结

view 的绘制流程分为三个阶段，测量、布局和绘制，分别确定了一个 View 的大小、位置和内容。ViewRootImpl 为总控，DecorView 为中介，将这些事件提交给 ViewTree ，ViewTree 也可以通过 ViewParent 接口将请求传递给 ViewRootImpl ，形成一个数据流通圈。