---
title: Android Handler 机制实现
date: 2018-04-30
tags:
	- android
---



Handler 机制是 Android 独有的一套切换线程的方式，Java 中并没有类似的东西（毕竟 Android 是以消息驱动的，Java 不是），那么是否可以在 Java 中实现类似的一套机制，下面先对 Android 中 Hanlder 机制使用到的东西进行分析：

## Handler 机制分析

前面从 Andorid 源码的角度了解了 Handler 机制的实现：[Andorid Handler 机制](/Android Handler 机制)，知道了 Handler 机制实现的一个主要方法就是 MessageQueue 中的 native 层方法`nativePollOnce(long, int)`和`nativeWake(long)`方法，完成了对当前线程的堵塞和唤醒，那么考虑到 Java 中线程操作相关的功能，也有相应的线程等待与唤醒方法`wait()`和`notify() notifyAll()`，可以使用这两个方法完成类似的功能。由此可以看出，这很像是一个生产者-消费者问题，`MessageQueue:enqueueMessage(Message, long)`充当着生产者，向消息队列中添加消息，并在合适的时候唤醒消费者，`MessageQueue:next()`充当着消费者，在没有消息可以处理的时候会陷入等待。

### Wait & Notify

`wait()`和`notify()`方法是 Object 类的方法，也就是说所有的实例都可以调用这两个方法，首先看它们的使用：

#### wait

`wait()`有三种调用方法：

```java
// 进入休眠，且不会自动返回
wait();

// 进入休眠，millis 毫秒后返回
wait(long millis);

// 进入休眠，millis 毫秒 + nanos 纳秒后返回
wait(long millis, int nanos);
```

当 wait 的等待时长为 0 的时候，也只能在被唤醒时才能返回。

#### notify

`notify()`有两种调用：

```java
// 唤醒一个在等待当前对象的线程，即曾调用过当前对象的 wait 方法的线程
notify();

// 唤醒所有等待当前对象的线程
notifyAll();
```

由此可见，wait 、 notify 就构成了消费者-生产者问题的两个动作，但是在生产者-消费者问题中，还需要考虑到线程的同步问题，当有两个线程同时在调用一个对象的这个方法的时候，必须要确定这两个方法到底谁先执行，而不是等待系统的调度才确定谁先执行，因为这个过程中涉及到唤醒的操作，如果`notify()`因为调度的原因使得其在`wait()`之前执行了，那么线程就会进入无止境的等待中去。所以，对`wait()`和`notify()`的调用都需要在`synchronized`代码段中执行，并且，`synchronized`锁住的对象应该就是需要调用`wait()`和`notify()`方法的对象。否则就会抛出`java.lang.IllegalMonitorStateException`异常。

```java
public void test1() {
    Object obj = new Object();
    
    obj.wait();
    obj.notify();
}

public void test2() {
    Object obj = new Object();
    Object lock = new Object();
    synchronized (lock) {
        obj.wait();
        obj.notify();
    }
}

public void test3() {
    Object obj;
    synchronized (obj) {
        obj.wait();
        obj.notify();
    }
}
```

上面的`test1()`和`test2()`方法都会抛出异常，只有`test3()`才是正确的使用方式。在使用 Java 独立实现 Handler 机制的时候，会采用上述的使用方法，使用`wait()`和`notify()`替换`nativePollOnce()`和`nativeWake()`。

另外，Android 使用 Handler 机制是为了线程的切换，在使用这套机制的前提是有一个主线程，多个其他线程，使用 Handler 发送消息要么是在其他线程，要么是在当前线程的「工作状态」中，在 ActivityThread 中调用了主线程 Looper 的`loop()`方法，我们知道这个方法会进入一个无限循环，消息处理也是在这个循环下完成的，所以所谓的主线程的工作状态，就是说在这之后所有在主线程中执行的代码都是通过`Handler:dispatchMessage(Message)`这个方法作为入口进而执行各种功能的，所以可以确定，Looper 处理的第一个消息一定是通过其他线程发送的，那么在本次的实现过程中，会通过开启一个新线程的方式接受用户输入并发送消息，主线程中则负责接收消息并处理。

## HANDLER 机制实现

代码需要实现 Message 、 Handler 、 MessageQueue 、 Looper 这四个类：

`os.Message`

```java
package os;

/**
 * @User: wuruoye
 * @Date: 2018/4/30 11:51
 * @Descrition:
 */
public class Message {
    Handler target;
    Message next;
    long when;

    public int what;
    public int arg1;
    public int arg2;
    public Object obj;
    public Runnable callback;

    private static Message mMessage;

    private Message() {}

    public static Message obtain() {
        synchronized (Message.class) {
            if (mMessage == null) {
                return new Message();
            }else {
                Message m = mMessage;
                mMessage = m.next;
                return m;
            }
        }
    }

    public static Message obtain(Handler handler) {
        Message m = obtain();
        m.target = handler;
        return m;
    }

    public static Message obtain(Runnable runnable) {
        Message m = obtain();
        m.callback = runnable;
        return m;
    }

    public static Message obtain(int what) {
        Message m = obtain();
        m.what = what;
        return m;
    }

    public void recycle() {
        synchronized (Message.class) {
            this.next = mMessage;
            mMessage = this;
        }
    }
}
```

`os.Handler`

```java
package os;

/**
 * @User: wuruoye
 * @Date: 2018/4/30 11:52
 * @Descrition:
 */
public class Handler {
    private MessageQueue mQueue;
    private Looper mLooper;

    public Handler() {
        mLooper = Looper.myLooper();
        mQueue = mLooper.getQueue();
    }

    public void post(Runnable runnable) {
        sendMessageAt(Message.obtain(runnable), System.currentTimeMillis());
    }

    public void sendMessage(Message message) {
        sendMessageAt(message, System.currentTimeMillis());
    }

    public void sendMessageDelay(Message message, long delay) {
        sendMessageAt(message, System.currentTimeMillis() + delay);
    }

    public void sendMessageAt(Message message, long timeMillis) {
        message.when = timeMillis;
        enqueueMessage(message);
    }

    private void enqueueMessage(Message message) {
        message.target = this;
        mQueue.enqueueMessage(message);
    }

    public void dispatchMessage(Message message) {
        if (message.callback != null) {
            message.callback.run();
        }else {
            handleMessage(message);
        }
    }

    public void handleMessage(Message message) {}
}
```

`os.MessageQueue`

```java
package os;

import java.util.LinkedList;
import java.util.List;

/**
 * @User: wuruoye
 * @Date: 2018/4/30 11:52
 * @Descrition:
 */
public class MessageQueue {
    private List<IdleHandler> mIdleHandlerList = new LinkedList<>();
    private Message mMessage;

    private boolean mBlock;
    private boolean mQuiting;

    public void enqueueMessage(Message message) {
        long when = message.when;

        synchronized (this) {
            if (mMessage == null || when == 0 || when < mMessage.when) {
                message.next = mMessage;
                mMessage = message;
            }else {
                Message current = mMessage, prev;
                do {
                    prev = current;
                    current = current.next;
                }while (current != null && when > current.when);

                prev.next = message;
                message.next = current;
            }

            if (mBlock) {
                notify();
                mBlock = false;
            }
        }
    }

    public Message next() throws InterruptedException {
        int pendingIdleHandlerCount = -1;
        long timeoutMillis = 0;

        for (;;) {
            synchronized (this) {
                if (mQuiting) {
                    return null;
                }

                if (timeoutMillis > 0) {
                    wait(timeoutMillis);
                }

                Message message = mMessage;
                long now = System.currentTimeMillis();

                if (message == null) {
                    mBlock = true;
                    wait();
                }else {
                    if (message.when <= now) {
                        mMessage = message.next;
                        return message;
                    }else {
                        timeoutMillis = message.when - now;
                    }

                    if (pendingIdleHandlerCount < 0 && (mMessage == null ||
                            now < mMessage.when)) {
                            pendingIdleHandlerCount = mIdleHandlerList.size();
                    }

                    if (pendingIdleHandlerCount <= 0) {
                        mBlock = true;
                        continue;
                    }

                    IdleHandler[] idleHandlers = new IdleHandler[pendingIdleHandlerCount];
                    idleHandlers = mIdleHandlerList.toArray(idleHandlers);

                    for (int i = 0; i < pendingIdleHandlerCount; i++) {
                        IdleHandler idle = idleHandlers[i];
                        idleHandlers[i] = null;

                        if (!idle.doInIdle()) {
                            mIdleHandlerList.remove(idle);
                        }
                    }

                    pendingIdleHandlerCount = 0;
                    timeoutMillis = 0;
                }
            }
        }
    }

    public void addIdleHandler(IdleHandler idleHandler) {
        synchronized (this) {
            mIdleHandlerList.add(idleHandler);
        }
    }

    public void quit() {
        mQuiting = true;
        synchronized (this) {
            Message message;
            while (mMessage != null) {
                message = mMessage;
                mMessage = message.next;
                message.recycle();
            }
        }
    }

    public static interface IdleHandler {
        boolean doInIdle();
    }
}
```

`os.Looper`

```java
package os;

/**
 * @User: wuruoye
 * @Date: 2018/4/30 13:29
 * @Descrition:
 */
public class Looper {

    private static ThreadLocal<Looper> sLooper = new ThreadLocal<>();

    private static Looper sMainLooper;

    private MessageQueue mQueue;

    public static void prepareMain() throws Exception {
        prepare();
        sMainLooper = myLooper();
    }

    public static void prepare() throws Exception {
        Looper looper = sLooper.get();
        if (looper != null) {
            throw new Exception("one looper can only prepare once.");
        }
        looper = new Looper();
        looper.mQueue = new MessageQueue();
        sLooper.set(looper);
    }

    public static Looper myLooper() {
        return sLooper.get();
    }

    public static void loop() throws Exception {
        Looper looper = myLooper();
        if (looper == null) {
            throw new Exception("looper is null, please prepare() first.");
        }

        for (;;) {
            Message message = looper.mQueue.next();

            if (message == null) {
                return;
            }

            if (message.target == null) {
                throw new Exception("the target of os.Message must not be null.");
            }else {
                message.target.dispatchMessage(message);
            }

            message.recycle();
        }
    }

    public MessageQueue getQueue() {
        return mQueue;
    }
}
```

`Main`

```java
import os.Handler;
import os.Looper;
import os.Message;

import java.util.Scanner;

/**
 * @User: wuruoye
 * @Date: 2018/4/30 14:17
 * @Descrition:
 */
public class Main {
    public static void main(String[] args) throws Exception {
        Scanner scanner = new Scanner(System.in);

        Looper.prepareMain();
        Handler handler = new Handler() {
            @Override
            public void handleMessage(Message message) {
                System.out.println("output " + Thread.currentThread().getName());
                System.out.println(message.what);
            }
        };
        new Thread(() -> {
            while (scanner.hasNext()) {
                int what = 0;
                long delay = 0;
                try {
                    what = scanner.nextInt();
                    delay = scanner.nextLong();
                } catch (Exception e) {
                    System.out.println("end");
                    System.exit(0);
                }
                System.out.println("input " + Thread.currentThread().getName());
                handler.sendMessageDelay(Message.obtain(what), delay);
            }
        }).start();

        Looper.loop();
    }
}
```

