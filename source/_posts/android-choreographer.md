---
title: Android Choreographer 简述
date: 2019-03-20 14:30
tags:
	- android
---

在 [Android 属性动画](../android-property-animation) 中讲到，ValueAnimator 的工作需要使用到 Choreographer 类，这个类负责的不只是与动画相关的功能，准确来说，它负责协调动画、输入和绘制这三种事件的工作。在这个类内部保存了回调队列数组 mCallbackQueues ，每一个队列存放着一个类型的回调对象，保存着下一个时间脉冲到达时要做的所有事情，在 Choreographer 中共有四种回调类型，分别是：

1.  INPUT，用户输入事件，如触摸、按键等
2.  ANIMATION，动画
3.  TRAVERSAL，ViewTree 绘制
4.  COMMIT

当一个时间脉冲到达时，Choreographer 会执行所有的回调，其执行顺序也就是上述的顺序。其本质其实也是一种类似于 Looper-Handler 的结构，这些事件都遵循着一个顺序：提交、等待、执行。但是与 Looper-Handler 不同的是，所有被提交的事件并不是直接执行，而是需要等到下一次脉冲到达时才会执行，也就是下一帧到达时一起执行，如果每秒有 60 个脉冲会到达，就意味着当前系统的刷新率为 60Hz 。下面从提交一个回调开始，看看这个流程如何运作。

### `postFrameCallback()`

```java
public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

提交一个事件，并将其加入回调队列 mCallbackQueues ，将事件包装成了一个 CallbackRecord 对象，之所以会对其进行包装，是因为提交的事件有两种类型：FrameCallback 和 Runnable ，前者是动画事件，它的执行方法是`doFrame(long)`，后者是通用型事件，执行方法为`run()`。

提交之后，先根据当前回调的执行时间，选择执行时间（使用 Handler 提供延时功能）。最终都会调用`scheduleFrameLocked()`方法。

### `scheduleFrameLocked()`

```java
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame on vsync.");
            }
            // If running on the Looper thread, then schedule the vsync immediately,
            // otherwise post a message to schedule the vsync from the UI thread
            // as soon as possible.
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
            if (DEBUG_FRAMES) {
                Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
            }
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}
```

在这个方法中，根据 USE_VSYNC 的值选择一种延时方式。VSYNC 是一种防止画面撕裂的技术，大致就是在屏幕刷新率和系统刷新率不一致的时候通过降低刷新率的方式保持画面的完整性，具体参见 [什么是 VSync](https://blog.csdn.net/zzqhost/article/details/7785376) ，如果不使用 VSYNC ，则直接使用 Android 的 Handler 机制，在延时一段时间之后调用`doFrame()`方法。而如果是 VSYNC 方法的话，会使用一些系统底层的相关功能。

### `scheduleVsyncLocked()`

```java
private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}

// DisplayEventReceiver.class
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

这里会调用 native 方法，调用这个方法之后，会在下一帧到达时调用它的`onVsync()`方法，其实现在子类 FrameDisplayEventReceiver 中。

### `onVsync()`

```java
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    // Post the vsync event to the Handler.
    // The idea is to prevent incoming vsync events from completely starving
    // the message queue.  If there are no messages in the queue with timestamps
    // earlier than the frame time, then the vsync event will be processed immediately.
    // Otherwise, messages that predate the vsync event will be handled first.
    long now = System.nanoTime();
    if (timestampNanos > now) {
        Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                + " ms in the future!  Check that graphics HAL is generating vsync "
                + "timestamps using the correct timebase.");
        timestampNanos = now;
    }
    if (mHavePendingVsync) {
        Log.w(TAG, "Already have a pending vsync event.  There should only be "
                + "one at a time.");
    } else {
        mHavePendingVsync = true;
    }
    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}

public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
}
```

在`onVsync()`方法中首先使用 Handler 进行线程调整，然后就会调用`doFrame()`方法。

与不使用 VSYNC 的方法相比，它们之间唯一的不同其实就是在这里，后者仅仅使用 Handler 发送一个延时任务去调用`doFrame()`方法而没有考虑刷新率之间的统一性，而 VSYNC 则是调用了系统底层的方法，做了一些刷新率方面的优化才回调`doFrame()`方法。

### `doFrame()`

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    // ...
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
        
        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        
        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        
        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

在这个方法中，通过调用四个`doCallbacks()`方法，分别执行了 INPUT、ANIMATION、TRAVERSAL 和 COMMIT 的回调事件，其执行顺序也就如上所列。这里可以看到，第三位才是执行 TRAVERSAL 事件，对应着 ViewTree 的绘制，那么如果上面两种事件中发生了执行事件过长的情况，就会导致视图刷新太慢，在用户看来就是卡顿现象。

### `doCallbacks()`

```java
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        // We use "now" to determine when callbacks become due because it's possible
        // for earlier processing phases in a frame to post callbacks that should run
        // in a following phase, such as an input event that causes an animation to start.
        final long now = System.nanoTime();
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;
        }
        mCallbacksRunning = true;
        // Update the frame time if necessary when committing the frame.
        // We only update the frame time if we are more than 2 frames late reaching
        // the commit phase.  This ensures that the frame time which is observed by the
        // callbacks will always increase from one frame to the next and never repeat.
        // We never want the next frame's starting frame time to end up being less than
        // or equal to the previous frame's commit frame time.  Keep in mind that the
        // next frame has most likely already been scheduled by now so we play it
        // safe by ensuring the commit time is always at least one frame behind.
        if (callbackType == Choreographer.CALLBACK_COMMIT) {
            final long jitterNanos = now - frameTimeNanos;
            Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
            if (jitterNanos >= 2 * mFrameIntervalNanos) {
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                        + mFrameIntervalNanos;
                
                frameTimeNanos = now - lastFrameOffset;
                mLastFrameTimeNanos = frameTimeNanos;
            }
        }
    }
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            
            c.run(frameTimeNanos);
        }
    } finally {
        synchronized (mLock) {
            mCallbacksRunning = false;
            do {
                final CallbackRecord next = callbacks.next;
                recycleCallbackLocked(callbacks);
                callbacks = next;
            } while (callbacks != null);
        }
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

首先根据事件类型选择一个合适的队列，然后按照队列的添加顺序执行所有事件回调，最后回收。`c.run()`方法，对应的实现为：

```java
public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
        ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
        ((Runnable)action).run();
    }
}
```

在这里执行最终的回调，然后 Choreographer 的工作基本完成，剩余的工作由工作主体接续完成，并且在工作的过程中，会继续向 Choreographer 提交下一帧的事件，等待调度。