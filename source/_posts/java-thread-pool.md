---
title: Java 线程池介绍
date: 2018-05-01
tags:
	- java
---

## 线程

在开发应用程序的时候，是用到线程的地方很多，一般情况下应用程序都会有一个与用户交互的界面，而在用户操作程序的过程中，肯定会经常执行到一些耗时操作，如网络请求、文件处理等，如果这些操作与处理用户交互的工作处于一个线程中，肯定会导致线程堵塞，在用户看来，就是界面卡顿了，这种情况肯定会造成用户的不满。

在 Android 应用中，更是不允许这种情况的发生，在 Android 4.0 之后，直接规定不能在主线程中进行网络请求，另外，如果主线程堵塞过长时间，就会报错 ANR（Application Not Ressponding），所以，多线程的使用变得更加频繁。

如果要开启一个子线程用于执行某些耗时工作，最简单的方法就是`new Thread(Runnable).start()`，就可以开启一个线程执行 Runnable 里面的`run()`方法了，这里有一点区分就是，如果直接调用`Thread:run()`方法，只会在当前线程执行这段代码，而不是开启新线程。

或者是继承 Thread 实现一个子类，在`run()`方法里面，写上要在子线程中执行的代码，如`android.os.HandlerThread`，它继承了 Thread 类，并在这个线程中使用了 Handler 机制，在`run()`方法里调用了`Looper:loop()`方法，如此一来，就可以通过调用`Handler:post()`和`Handler:sendMessage()`方法随时在这个线程中执行工作了，一般情况下，可用于某些连续性的操作，线程在执行了一个任务之后，不会直接销毁，而是继续等下一个工作的到来，只有调用了`HandlerThread:quit()`方法，线程才会销毁，在某些时候实现了线程的复用，如在大量频发的网络请求的时候，每次请求创建一个线程再随即销毁显然是不合理的，那么就可以使用这种方法。

HandlerThread 虽然实现了线程复用，但是有两个问题：

1. 只能用于一个线程的复用，如果一时间工作太多的话，会造成比较长时间的堵塞
2. 本身是使用 Handler 机制实现的，开销比较大

于是为了解决线程复用的问题，又有了线程池这个东西，简单来说，就是线程池里面拥有多个线程，用户需要在子线程中执行相关工作的时候，只需要将工作提交到线程池，然后线程池会将任务下发给它里面的某一个线程处理，如此便可完成多个线程的调度等问题，并且其内部实现原理也只是线程有关的方法，开销也会相应降低。

## 线程池

在`java.util.concurrent`包里，提供了一些关于并发编程需要的类，如 Executor 、ThreadPoolExecutor 等，继承树如下：

![](http://blog-1251826226.coscd.myqcloud.com/thread_pool_tree.jpg)

Executor 是一个接口，只有一个方法`exec(Runnable)`，其中关于线程池的主要实现是 ThreadPoolExecutor 类，对于一个线程池来说，有几个相关概念：

1. corePoolSize

   线程池中核心线程数量，当已有的线程数量小于这个数字的时候，每向线程池中添加任务的时候，都会创建一个线程执行

2. maximumPoolSize

   线程池中所能存放的最大任务数。如果当前线程池中的任务达到这个数目，就会拒绝添加这个任务。

3. keepAliveTime

   一个线程存活的时间，从没有要执行的任务开始等待，如果等待了这么长的时间之后还没有得到任务，就记为超时，对于核心线程，会继续等待，对于非核心线程，会被销毁。

4. BlockingQueue

   线程池中用于存放待处理任务的队列，多个线程共用此任务队列，并从中取出任务，在取任务的时候可能会进入堵塞状态，直到添加进来任务才会返回。

5. Worker

   用于执行任务的类，与线程是一一对应的，是线程池中线程的表现形式，线程池中并没有对线程的直接操作，而是以操作 Worker 的方式，每次创建 Worker 的同时也都创建了一个 Thread 。实现了 Runnable 接口，运行的时候会持续从 BlockingQueue 中取出任务执行。


线程池 ThreadPoolExecutor 工作的基本流程如下图：

![](http://blog-1251826226.coscd.myqcloud.com/thread_pool_work.jpg)

可以看到，线程池的工作原理其实与 Handler 机制的工作原理很像，这当中 Worker 充当了 Looper 的作用，BlockingQueue 充当了 MessageQueue ，并通过 ThreadPoolExecutor 发送 Runnable ，唯一的区别可能就是，线程池中增加了对 Worker 的管理，并且多个线程共用同一个 BlockingQueue ，整个 ThreadPoolExecutor 类就是前面讲的[Android Handler 机制](/Android Handler 机制)变体，精简了 Message 这个类，只保留了 Runnable ，但是扩充成了多个线程共同工作的方式。

首先对于线程池 ThreadPollExecutor 的构造来看，这是它的一个构造方法：

`java.util.concurrent`

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

上面有些字段已经说过，都是一些配置性的问题，构造方法的最后三个参数，分别是堵塞队列、线程工厂、拒绝策略。

### 堵塞队列 BlockingQueue

BlockingQueue 是一个接口，继承自 Queue ，并添加了用于获取的方法`take()`和`poll()`，这两个方法能够堵塞线程，BlockingQueue 有多个实现：

1. ArrayBlockingQueue

   内部使用数组实现，固定大小，采用头尾两个指针以循环表的方式。

2. DelayWorkerQueue

   采用数组实现，可变大小，并且操作的是一个 RunnableScheduledFuture 对象，有延迟功能。

3. LinkedBlockingQueue

   链表实现，固定容量，可自己设置（默认 Integer.MAX_VALU ）

4. LinkedBlockingDeque

   双向链表实现，固定容量，可从头部或尾部插入、取出

5. LinkedTransferQueue

   链表实现，通过入队操作（放操作、取操作）的方式完成数据的操作。

   - 放操作

     如果当前队列中有取操作正在等待，直接将数据交由取操作，然后唤醒取操作所在线程，否则入队

   - 取操作

     如果当前队列中有放操作，取出然后返回，否则入队取操作，线程进入等待状态

   使用这种方法能够提高队列的效率。

6. PriorityBlockingQueue

   数组实现，可变大小，在队列中按照优先级排列

7. SynchronousQueue

   内部通过 TransferQueue 实现，实现方法与 LinkedTransferQueue 类似，但是没有缓冲区，也就是说，对于放操作来说，如果没有对应的取操作，在入队的同时也会堵塞当前线程

BlockingQueue 的实现类比较多，使用了不同的策略完成了「堵塞队列」这一功能，基本的使用就是：

1. 某线程向队列中添加任务

   一般情况下，某线程向队列中提交任务之后，将任务添加到 BlockingQueue 的缓冲队列之后就会立刻返回（大部分实现都是如此），不过也有可能提交任务的线程也会被堵塞，直到任务被处理了才返回（SynchronousQueue）。然后在新的任务来到之后，会调用唤醒方法唤醒那些等待任务的线程。

2. 某线程从队列中取出任务

   一般情况下，如果当前队列没有任务分配给这个线程，就会堵塞这个线程，直到有新的任务添加进来的时候，才会唤醒被它堵住的线程，然后其中某些获取到了任务，完成返回，另外没有分配到任务的，继续等待。

堵塞队列提供的线程堵塞与唤醒等一系列的方法，完成了线程池中各任务的有序执行，线程池中的每个线程不断地执行任务与等待任务。在 线程池中一系列线程工作的过程中，BlockingQueue 无疑完成了很重要的工作，因为线程的堵塞和唤醒都是在 BlickingQueue 中完成的，那么在 ThreadPoolExecutor 中操纵每个线程工作的时候，只需要调用`poll()`和`take()`和`offer()`方法，就可以直接获取和添加任务。

### 线程工厂 ThreadFactory

ThreadFactory 也是一个接口，定义了创建一个新线程的方法，它的作用主要是在创建一个 Worker 的时候，以什么样的方式创建一个新的线程，如设置一个线程为守护线程，或者按照某个编号给线程命名等，这些都可以通过自定义一个 ThreadFactory 完成。

### 拒绝策略 RejectedExecutionHandler

RejectedExecutionHandler 也是一个接口，定义了一个拒绝执行某个任务时调用的方法`rejectedExecution(Runnable, ThreadPoolExecutor)`，用于当一个线程池不能接受这个任务的时候回调的方法，可以通过自定义一个 RejectedExecutionHandler 的方式处理一个 Runnable 被 ThreadPoolExecutor 拒绝之后的操作，如换个方式执行 Runnable 或者打印一条日志等。

## 线程池工作过程

### 添加任务

构造完成一个线程池之后，就可以调用`ThreadPoolExecutor:execute(Runnable)`方法执行一个任务：

`java.util.concurrent.ThreadPoolExecutor`

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * 处理一个新任务的 3 个步骤：
     *
     * 1. 如果当前运行的线程数小于 corePoolSize ，开启一个新线程并将这个 Runnable
     * 交由线程作为线程的第一个任务。这个方法并没有线程锁，所以在 addWorker 方法中
     * 在加锁的情况下会再次会当前 worker 的数量和 线程池状态作检测，如果不能添加新的 
     * worker ，就会返回 false 。继续接下来的动作，否则如果添加 worker 成功了，就
     * 直接返回本方法。
     *
     * 2. 如果任务成功入队了，则需要再次检测是否需要新添加一个线程（如在入队的过程中
     * 有线程被销毁了）或者在这个过程中线程池被关闭了。所以这里需要再次检测是否需要回滚
     * （当线程池已经关闭）或者开启一个新线程（如果当前没有线程在工作）。
     *
     * 3. 如果不能将任务入队，表示此时堵塞队列已经满了，这时需要开启非核心线程执行任务，
     * 如果这次操作再失败，表示最大能够存在的线程数量也到了，就只能拒绝执行这个任务了。
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

以上就是当一个任务提交到线程池的时候线程池要做的判断与下一步的操作，这个方法里多次使用`ctl.get()`，这个 ctl 是 AtomicInteger 的实例，也就是说，这个整型数据能保证，它的读写操作是原子性的，那么就可以确保在多线程中，对它的读写不会因为线程间的同步问题造成影响，而这个字段存放的有两种数据，其高三位存放的是当前线程池的状态，其余位则存放线程池中 worker 的数量。线程的状态有五种：

```java
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

然后使用下面的方法解析整形得到对应的状态和 worker 数量，在之后的操作中，这个变量使用的地方也很多。

在这个方法里，Runnable 的出口一共有三个方向：

1. 添加 worker 执行这个 Runnable
2. 将 Runnable 入队
3. 拒绝处理 Runnable

上面已经说过拒绝一个任务的处理方法了，通过调用 RejectedExecutionHandler 的方法完成。将任务入队，调用的是`BlockingQueue:offer(Runnable)`，因为 BlockingQueue 是一个接口（上面介绍过），所以它的入队方法也是有很多种实现的，大致的功能都是将 Runnable 入队，并在需要的时候唤醒某个工作线程。

### 添加 worker

添加完任务的下一步就是添加 worker ，worker 是线程的封装，代表每一个正在运行的线程。添加 Worker 调用的是`addWorker(Runnable, boolean)`，第一个参数表示要添加的任务，第二个参数标识添加的是否为核心线程（决定线程数量的边界），然后在这个方法里，首先是对线程池状态的判断，然后判断线程数量等，最后创建 Worker 并使其开始工作。 Worker 本身是一个 Runnable ，然后内部有一个 Thread 变量，调用这个 Thread 的`start()`的方法的之后，就会执行 Worker 的`run()`方法，而这个方法又会调用`ThreadPoolExecutor:runWorker(Worker)`方法：

`java.util.concurrent.ThreadPoolExecutor`

```java
/**
 * Main worker run loop.  Repeatedly gets tasks from queue and
 * executes them, while coping with a number of issues:
 *
 * 1. We may start out with an initial task, in which case we
 * don't need to get the first one. Otherwise, as long as pool is
 * running, we get tasks from getTask. If it returns null then the
 * worker exits due to changed pool state or configuration
 * parameters.  Other exits result from exception throws in
 * external code, in which case completedAbruptly holds, which
 * usually leads processWorkerExit to replace this thread.
 *
 * 2. Before running any task, the lock is acquired to prevent
 * other pool interrupts while the task is executing, and then we
 * ensure that unless pool is stopping, this thread does not have
 * its interrupt set.
 *
 * 3. Each task run is preceded by a call to beforeExecute, which
 * might throw an exception, in which case we cause thread to die
 * (breaking loop with completedAbruptly true) without processing
 * the task.
 *
 * 4. Assuming beforeExecute completes normally, we run the task,
 * gathering any of its thrown exceptions to send to afterExecute.
 * We separately handle RuntimeException, Error (both of which the
 * specs guarantee that we trap) and arbitrary Throwables.
 * Because we cannot rethrow Throwables within Runnable.run, we
 * wrap them within Errors on the way out (to the thread's
 * UncaughtExceptionHandler).  Any thrown exception also
 * conservatively causes thread to die.
 *
 * 5. After task.run completes, we call afterExecute, which may
 * also throw an exception, which will also cause thread to
 * die. According to JLS Sec 14.20, this exception is the one that
 * will be in effect even if task.run throws.
 *
 * The net effect of the exception mechanics is that afterExecute
 * and the thread's UncaughtExceptionHandler have as accurate
 * information as we can provide about any problems encountered by
 * user code.
 *
 * @param w the worker
 */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

这里使用 while 循环，使用到了`getTask()`获取任务，这个方法可能会堵塞，如果 task 为空就会退出本次循环，并尝试结束当前线程，如果不为空，就会执行任务并继续请求获取任务。这种工作模式简直跟`Looper.loop()`方法一模一样，同样是堵塞获取，然后无限循环，然后条件退出循环。在正常情况下，此循环退出的情况有两个：

1. `getTask()`返回 null 
2. 在这个执行过程抛出异常

结束循环之后，线程池会尝试关闭当前的线程，使用`processWorkerExit(Worker, boolean)`方法，还会有一些其他的处理，如保留最少一个 worker 等。

### 获取任务

只有获取到任务的时候 worker 才能工作，获取任务使用的是`getTask()`方法，这个方法也跟`MessageQueue:next()`方法很相像，从堵塞队列 BlockingQueue 中获取，这里有一个区分，对于核心线程来说，会一直处于等待状态，对于非核心线程，如果等待了 keepAliveTime 时间之后还没有获取到任务，就会返回一个 null 。区分是否为核心线程的方法是`boolean timed = allowCoreTimeout || wc > corePoolSize`，当前的 worker 数量大于核心线程数量，就认为线程是非核心的。

在任务不充足的情况下，`getTask()`使线程陷入等待状态，并且会在添加任务的时候唤醒一个线程。从而完成线程池中线程的管理。

### 扩展

线程池 ThreadPoolExecutor 只能通过`execute(Runnable)`方法执行 Runnable 对象，却不能操作后来出现的 Callable 对象，Runnable 与 Callable 是两个功能类似的接口，不过 Callable 的`call()`方法能够返回一个值，这就是他们之间的区别，所以后来 ThreadPoolExecutor 有了一个子类 ScheduledThreadPoolExecutor ，ScheduledThreadPoolExecutor 没有直接使用 Runnable 作为任务入队，而是使用 ScheduledFutureTask 类封装了 Runnable 或者是 Callable ，将这个作为任务单位提交到 BlockingQueue ，取出来执行的也是这个类。本质上 ScheduledFutureTask 也是一个 Runnable ，所以它可以直接提交给 ThreadPoolExecutor 的 BlockingQueue ，不过它在任务执行的过程中对 Runnable 和 Callable 作了额外的操作，先看下 ScheduledFutureTask 的继承树：

![](http://blog-1251826226.coscd.myqcloud.com/scheduled_future_task_tree.jpg)

Future 是一个用于获取异步计算结果的接口，它提供了`get()`方法用户得到结果，这个方法可能会堵塞线程，在 FutureTask 对这个接口作了具体的实现，FutureTask 实现的基本原理是：通过传入 Callable 或 Runnable （Runnable 会被一层 Callable 封装成为 Callable ）作为它的成员变量，自己则作为一个 Runnable 作为任务交由 BlockingQueue ，然后在工作线程调用到自己的`run()`方法的时候，再调用它内部的 Callable 对象的`call()`方法，并将`call()`返回的结果保存起来，然后再使用一个整型标记当前的状态。在使用某一线程调用`get()`方法的时候，会对当前的状态检测，如果已经得到结果，就直接返回保存的结果，否则就会使这个线程进入等待状态。另一方面，当工作线程完成自己的任务，也就是`call()`返回结果之后，首先会更改当前的状态为完成状态，然后会唤醒被`get()`方法堵塞的线程。FutureTask 的状态有七个，分别是：

1. NEW
2. COMPLETING
3. NORMAL
4. EXECPTIONAL
5. CANCELED
6. INTERRUPTING
7. INTERRUPTED

线程状态转换的可能为以下四种之一：

1. NEW -> COMPLETING -> NORMAL
2. NEW -> COMPLETING -> EXCEPTIONAL
3. NEW -> CANCELLED
4. NEW -> INTERRUPTING -> INTERRUPTED

但是 ScheduleThreadPoolExecutor 还不是直接使用 FutureTask ，而是它的子类 ScheduledFutureTask ，这个类还另外实现了 RunnableScheduledFuture 接口，增加了定期执行的功能，不过主要的一些操作还是父类实现的。

ScheduledThreadPoolExecutor 可以执行 Runnable 和 Callable ，它们都会进入`schedule()`方法，使用 ScheduledFutureTask 将其包装，再提交给线程池工作。与 ThreadPoolExecutor 提交任务的`execute(Runnable)`方法不同，ScheduledThreadPoolExecutor 使用了`delayedExecute()`方法提交任务，主要的区别就是，它不会直接创建一个携带初始任务的 Worker ，而是直接将任务加入到 BlockingQueue 中，然后再创建一个没有初始任务的 Woker 加入到工作中，这样导致的区别就是，任务会根据自己的延迟时间（因为 ScheduleFutureTask 实现了 Delayed接口）在队列中排序，经过 DelayedWorkerQueue 处理，完了之后再根据这个时间依次选择一个任务执行，而不是上来就执行。

## 线程池使用

虽然线程池的核心实现是 ThreadPoolExecutor ，但是线程池还是有几个不同的种类的，而且线程池的构造参数太多，对于一些从简的人而言可能会有些不适，而 Executors 就是为了解决这个问题出现的，Executors 可以被看作为一个线程池的工厂类，它提供了众多的静态方法如`newFixedThreadPool(int)`用来生成线程池，如果没有比较多的定制性需求，如自定义 BlockingQueue 、自定义 RejectedExecutionHandler 等，使用这个类可以比较简单的生成不同的线程池。

线程池的使用也可以利用工厂类 Executors ，这个类提供了一些生成线程池的工厂方法，这些线程池共四种：

-   FixedThreadPool

    ```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    ```

    创建一个拥有固定线程数的线程池，当所有的线程都在运行时不接受新任务。

-   SignleThreadExecutor

    ```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    ```

    创建一个只有一个线程的线程池。

-   CachedThreadPool

    ```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    ```

    参数如上，这种线程池在没有任务的时候不会持续运行线程，一个线程的存活时间为 60s ，也就是说如果一个任务在另一个任务执行完的 60s 内被提交给线程池，线程池就会复用这个线程，超时就会将线程销毁。

-   ScheduledThreadPool

    ```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    ```

    创建一个核心线程数 corePoolSize ，最大线程数 Integer.MAX_VALUE 的线程池，与普通的 ThreadPoolExecutor 相比的话，就是可以延迟执行一个任务，通过`schedule()`方法。

## 总结

线程池为线程复用、大量频发异步请求提供了比较好的解决方案，并且有多个适应不同环境的子类，在高并发环境中使用比较广泛，开发应用的过程中，一些异步请求如 I/O 、 网络请求等都需要子线程来处理，如果这些工作的频率比较高的话，可以使用一个供全局使用的线程池，减少资源的消耗，并给以更快的响应。