---
title: Android Handle 机制
date: 2018-04-28 19:00
tags:
	- android
	- android 源码
---

## ANDROID 消息机制

Android 系统的运行采用消息机制，在主线程中所有的操作的执行都需要依靠 Hadnler 发送消息的方法来完成，因为在 ActivityThread 中创建了主线程的 Looper 并且调用了`Looper:loop()`方法，这个方法会堵塞当前线程（因为其内部有一个死循环），之后的所有操作的执行都是基于消息的机制来完成的。

在 Android 的消息机制中，主要完成任务的几个类为：Looper 、 Handler 、 Message 、 MessageQueue 。大致的流程为，Message 携带着要执行的操作信息，Handler 负责将 Message 传递到 MessageQueue ，MessageQueue 负责存储待处理的 Message ，Looper 负责从 MessageQueue 中取出 Message 并处理。这样就构成了一个完整的消息处理过程，使用这套机制的时候，利用 Handler 发送对应的消息就可以了。基本流程图如下所示：

![](http://blog-1251826226.coscd.myqcloud.com/handler_message.jpg)

接下来具体了解下这几个相关类：

### Message

>Defines a message containing a description and arbitrary data object that can besent to a {@link Handler}.  This object contains two extra int fields and anextra object field that allow you to not do allocations in many cases.

Message 作为信息的载体，它传递信息的时候有两种方式，一种是通过`what:int arg1:int arg2:int data:Bundle`这四个公有成员变量，一般是使用 what 作为操作码规定要执行的操作，然后使用后面的两个 int 和一个 Bundle 传递数据，而它们的使用规则就是，如果仅需要传递一到两个 int 就避免再使用 Bundle 传值。另一种则是通过携带一个 Runnable 对象，这个对象里面有需要执行的代码，在执行的时候，会直接调用`Runnable:run()`方法完成操作。

同时 Message 还有一个消息池机制，用于 Message 的循环利用，减少实例的频繁创建和回收，在 Looper 处理完一个 Message 的时候，会调用它的`recycleUnchecked()`方法，这个方法的工作就是重置 Message 之后将其加入消息池：

`android.os.Message`

```java
/**
 * Recycles a Message that may be in-use.
 * Used internally by the MessageQueue and Looper when disposing of queued Messages.
 */
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

然后需要使用一个 Message 的时候一般不会直接调用它的构造方法，而是使用`obtain()`方法，从消息池中取出一个实例：

`android.os.Message`

```java
/**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

可以看到，只有在消息池中已经没有可用消息的时候才会创建一个实例。

Message 是信息的载体，它本身并不包含动作性功能，Handler 负责 Message 的发送。

### Handler

Handler 的功能为发送 Message 和执行 Message 所携带的操作，对应于 Message 两种携带信息的方式，Handler 也有两种发送消息的方式，即`Handler:post(Runnable)`和`Handler:sendMessage(Message)`，当然前者也是先通过 Runnable 生成对应的 Message 然后再调用后者完成发送工作的，这里就权且看成两种。 Handler 通过 post 系列和 sendMessage 系列方法发送消息的时候，最终都会调用到`Handler:sendMessageAtTime(Messaeg, long)`。

`android.os.Handler`

```java
/**
 * Enqueue a message into the message queue after all pending messages
 * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
 * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
 * Time spent in deep sleep will add an additional delay to execution.
 * You will receive it in {@link #handleMessage}, in the thread attached
 * to this handler.
 * 
 * @param uptimeMillis The absolute time at which the message should be
 *         delivered, using the
 *         {@link android.os.SystemClock#uptimeMillis} time-base.
 *         
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

最后这个方法也会通过调用`MessageQueue:enqueueMessage(Message, long)`方法将 Message 添加到 MessageQueue 中去。这是 Handler 的发送消息功能。在 Looper 从 MessageQueue 中取出一个 Message 的时候，也会调用 Handler 的`dispatchMessage(Message)`方法完成 Message 的处理。

`android.os.Handler`

```java
/**
 * Handle system messages here.
 */
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}

/**
 * Subclasses must implement this to receive messages.
 */
public void handleMessage(Message msg) {
}
```

由这里就可以看到，如果 Message 携带的操作是以 Runnable 方式存在的，就会直接调用`Runnable:run()`，否则就会转而调用`handleMessage(Message)`，这个方法就是在使用 Handler 的时候最经常使用的方法，一般情况下，我们会继承 Handler 并重写 handleMessage 方法，在这个方法里面写上对 Message 的处理过程。

### MessageQueue

MessageQueue 作为 Message 的存储站，Handler 向其中发送消息，Looper 从其中取出消息，同时还要照顾到每个 Message 应该被处理的时间不同，以及涉及到当然没有 Message 可取的时候的线程等待，所以 MessageQueue 的功能是最复杂的。

先看一遍一个 Message 被放入 MessageQueue 直到被取出的过程，首先调用`enqueueMessage(Message, long)`方法将 Message 加入 MessageQueue ：

`android.os.MessageQueue`

```java
boolean enqueueMessage(Message msg, long when) {
    
    ...
    
    synchronized (this) {
        
        ...
        
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

这里完成了将 Message 加入到 MessageQueue ，可以看到，在这个队列中，它是一个按照时间存放的，所以可以将 MessageQueue 是一个优先级队列，时间越小优先性就越高，从而得以保证 Message 可以按照规定的时间顺序被处理。

Looper 通过`next()`方法从 MessageQueue 中取出一个 Message ，如果当前的 MessageQueue 已经被注销了，就会返回 null ，否则，当 MessageQueue 不为空且第一个 Message 的 when （即规定的 Message 应该被处理的时间）不大于当前时间的时候，这个 Message 就会被送给 Looper 处理，否则就会进入等待状态，这里的等待分为两种。

1. 当前 MessageQueue 中没有待处理的 Message ，进入长时间的休眠，需要主动唤醒，比如在`enqueueMessage(Message)`方法中执行唤醒操作
2. 当前 MessageQueue 队首的 Message 的 when 大于当前时间，则休眠一段时候，在 when 时间会自动唤醒

所以，MessageQueue 的 next 方法应该有很多时间都是处于堵塞状态的。

`android.os.MessageQueue`

```java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;
        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

在这个方法中，如果 MessageQueue 退出或者 mPtr 为 0 的时候，就会返回 null ，告诉 Looper 自己已经不再工作了。然后可以看到，这里面有一个`for(;;)`循环，在一次循环的过程中，会首先进入堵塞状态，堵塞的时间就是 nextPollTimeoutMillis ，默认是 0 。这里调用的一个 native 方法`nativePollOnce(long, int)`，第一个参数是当前 MessageQueue 的标识 mPtr ，第二个参数就是堵塞的时间，即会在这段时间之后返回，当传入的参数为 -1 的时候，会进入长时间堵塞，直到调用了`nativeWake(long)`才会返回。方法返回之后，就会取出队首的 Message ，如果这个消息的 when 大于当前时间，会执行`nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)`，如果当前队列中没有 Message ，就会置`nextPollTimeoutMillis = -1`，如果有可用的 Message ，就会将 Message 返回，结束当前的方法。另外 MessageQueue 中还存放了一个 IdleHandler 列表，会在 Looper 处于空闲期，即当前 MessageQueue 为空或者未到达 Message 的处理时间的时候逐个调用这些对象的`queueIdle()`方法，这些只会在第一次循环的时候调用，当进入新一轮循环之后，就会根据当前得到的 nextPollTimeoutMillis 的值决定休眠的时间。

MessageQueue 中因为使用了几个 native 方法，所以需要对这几个方法的作用有一定了解。与 native 相关的成员变量有 mPtr ，这个变量在 C/C++ 层次唯一标识了当前的 MessageQueue ，MessageQueue 调用的 native 方法通过 mPtr 找到对应的 MessageQueue 在 C/C++ 层代码的对应的对象，从而进行操作。这几个 native 方法分别为：

1. nativeInit

   调用 native 层的代码完成相应的 Looper 和 MessageQueue 的初始化

   `frameworks/base/core/jni/android_os_MessageQueue.cpp`

   ```c++
   static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
       NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
       if (!nativeMessageQueue) {
           jniThrowRuntimeException(env, "Unable to allocate native queue");
           return 0;
       }

       nativeMessageQueue->incStrong(env);
       return reinterpret_cast<jlong>(nativeMessageQueue);
   }
   ```

   这个方法返回了一个 long 类型的数值，可以看到，这里返回的就是 native 层的 NativeMessageQueue 的地址，这个地址在 MessageQueue 中使用了`long mPtr`存放，并且在相关操作的时候都会将 mPtr 作为参数传入，就是为了利用这个参数找到对应的 NativeMessageQueue 对象。

2. nativeDestroy

   调用 native 层代码完成对象销毁等

   `frameworks/base/core/jni/android_os_MessageQueue.cpp`

   ```c++
   static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
       NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
       nativeMessageQueue->decStrong(env);
   }
   ```

   NativeMessageQueue 的上层父类是 RefBase ，在它创建和销毁的时候都是调用了 RefBase 的`incStrong()`和`decStrong()`来完成的，

3. nativePollOnce

   调用 native 层代码执行线程堵塞

   `frameworks/base/core/jni/android_os_MessageQueue.cpp`

   ```c++
   static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
           jlong ptr, jint timeoutMillis) {
       NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
       nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
   }

   void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
       mPollEnv = env;
       mPollObj = pollObj;
       mLooper->pollOnce(timeoutMillis);
       mPollObj = NULL;
       mPollEnv = NULL;

       if (mExceptionObj) {
           env->Throw(mExceptionObj);
           env->DeleteLocalRef(mExceptionObj);
           mExceptionObj = NULL;
       }
   }
   ```

   Java 层通过调用 native 方法，在 native 层，调用到了`NativeMessageQueue:pollOnce()`，然后这个方法又调用了`Looper:pollOnce()`方法，在 Looper 中完成了线程堵塞的操作：

   `system/core/libutils/Looper.cpp`

   ```c++
   int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
       int result = 0;
       for (;;) {
           
           ...

           result = pollInner(timeoutMillis);
       }
   }

   int Looper::pollInner(int timeoutMillis) {
       
       ...

       struct epoll_event eventItems[EPOLL_MAX_EVENTS];
       // 这句代码调用了 epoll_wait 线程等待 timeoutMillis 时长
       int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

       ...
       
       return result;
   }
   ```

   epoll 是 linux 一个比较重要的概念，这里只当作一个线程堵塞功能看待。在这里，如果`timeoutMillis >= 0`，会在 timeoutMillis 时间后自动唤醒返回，如果`tiemoutMillis == -1`，就必须要调用唤醒程序才能返回，即下面的`nativeWake()`。

4. nativeWake

   调用 native 层代码完成线程唤醒

   在 MessageQueue 中会通过调用`nativeWake()`方法唤醒线程，这个方法一般在`enqueueMessage(Message, long)`中使用，其中使用到了 mBlock 标识记住线程是否处于堵塞状态来判断是否调用唤醒方法。

   `frameworks/base/core/jni/android_os_MessageQueue.cpp`

   ```c++
   static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
       NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
       nativeMessageQueue->wake();
   }

   void NativeMessageQueue::wake() {
       mLooper->wake();
   }
   ```

   `system/core/libutils/Looper.cpp`

   ```c++
   void Looper::wake() {
       
       ...

       uint64_t inc = 1;
       ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
       
       ...
   }
   ```

   这里的`write()`方法，完成了线程的唤醒。

5. nativeIsPolling

   调用 native 层代码获取当前状态（是否处于空闲状态），native 层的 Looper 会直接返回自身的是否空闲标识。

6. nativeSetFileDescriptorEvents

   调用 native 层代码，注册 FileDescriptorEvents 的监听等。

另一个是关于 IdleHander 的使用。由`next()`方法可以看到，在每次 Looper 调用 next 方法取 Message 的时候，如果需要休眠一段时间，就会执行一遍所有添加进来的`IdleHandler`的`queueIdle()`方法，也就是说，它会在当前线程处于空闲状态的时候执行，源码中对这个功能的使用地方不是很多，其中一个就是在 ActivityThread 中，通过调用`ActivityThread:scheduleGcIdler()`方法，往 MessageQueue 中添加了一个用于 GC 的 IdleHandler ，则可以在当前线程中没有消息要处理的时候执行一下 GC 操作，使得线程的使用率更高这样。其他的一些如果想要在线程间空闲的时候执行的操作也可以通过这种方法。在了解 Handler 机制的时候也可以使用这个功能，就可以知道什么时候会执行 next 方法，什么时候线程会空闲，进一步了解这种机制的 停-等 模式。

在 MessageQueue 中有一个成员变量 mNextBarrierToken ，对应的是 MessageQueue 添加消息栅栏的功能。在 MessageQueue 中，将 Message 分为三种：

1. 同步消息
2. 异步消息
3. 栅栏

使用 Handler 发送的是同步和异步消息，只需要通过`Message:setAsynchronous(boolean)`就可以设置，而栅栏则是使用`MessageQueue:postSyncBarrier()`添加到消息队列中的：

`android.os.MessageQueue`

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

可以看到，栅栏实际上就是 target 为 null 的消息，并且每个栅栏都有一个 token ，这个 toke 就是由 mNextBarrierToken 生成的，然后当`next()`方法中获取到栅栏类的消息的时候，会继续往下寻找异步消息进行处理，所有的同步消息都会被忽略，也就是说，「栅栏」的存在，阻止了同步消息的处理，唯有当栅栏被移除之后，同步消息才能得到处理，使用`removeSyncBarrier(int)`方法移除，这里的参数就是之前的 token ，MessageQueue 会根据这个 token 检索到对应的栅栏然后将其移出消息队列。

还有一个是关于 FileDescriptor 的一些操作。可以调用`MessageQueue:addOnFileDescriptorEventListener(FileDescriptor, int, OnFileDescriptorEventListener)`向注册监听，可对 I/O 事件进行监听，MessageQueue 再通过`nativeSetFileDescriptorEvents(long, int, int)`方法调用 native 层相关方法设置监听，然后再通过 native 调用方法`MessageQueue:dispatchEvents(int, int)`完成回调。

### Looper

终于，Message 经过在 MessageQueue 中的等待，被 Looper 取了出来准备执行相关操作。取 Message 的过程是在`loop()`方法中完成的，Looper 是一个线程间的单例，是 Handler 机制的基础，一个有 Looper 的线程才能使用消息机制工作，Looper 的构造方法如下：

`android.os.Looper`

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

在它构造方法中，创建了一个 MessageQueue 实例，并且 Handler 的构造方法也是需要依靠当前线程的 Looper 实例完成的：

`android.os.Handler`

```java
public Handler(Callback callback, boolean async) {
    
    ...
    
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

因为在主线程中系统的代码已经完成了主线程的 Looper 的初始化，所以在使用主线程的 Handler 机制的时候并不需要初始化 Looper ，但是如果在子线程中使用 Handler 的时候，就会报这种错误，另外，关于 Looper 的使用，当在一个线程中调用了`Looper:loop()`方法之后，这个方法之后的代码都执行不到了，只能通过 Handler 发送消息的方式工作，因为在`loop()`中有一个`for(;;)`循环，调用`quit()`方法才能退出这个循环，这样的话也就不能再使用 Handler 发送消息了。

`android.os.Looper`

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        
        ...
        
        msg.target.dispatchMessage(msg);
        
        ...
        
        msg.recycleUnchecked();
    }
}
```

在这个方法里，Looper 会一直调用`MessageQueue:next()`方法取下一个 Message ，如果 MessageQueue 返回了一个 null ，就表示退出这个循环，否则的话可以看到，Looper 会调用`Handler:dispatchMessage(Message)`方法完成 Message 的处理，并在最后将 Message 加入消息池中。
