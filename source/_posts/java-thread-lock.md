---
title: Java 线程锁
date: 2019-04-03 17:30
tags:
	- java
---

### 线程锁

线程锁是什么？首先要明确一系列的概念，在操作系统中，线程之间共享同一个进程的内存，所以再多线程环境中，经常会遇到多个线程同时操作同一个变量的情况，如果是读操作还好说，但是对于写操作，很可能会出现问题。这又涉及到操作的原子性问题，只有原子性的操作在多线程中才是安全的，如`x = 5`（直接赋值），而操作`x = y + 1`就不是原子性的，它在执行的过程中还要分三步：

1.  从 y 内存读出数值
2.  对这个数值 +1
3.  将数值写入到 x 内存

所以，如果两个线程同时执行`x = y + 1`操作，当线程 A 执行到第 2 步时，将 CPU 给到线程 B 执行，那么 B 也会再来一个上面的流程，当 B 执行完了之后会再执行 A 线程的第 3 步，但是 A 中的值还是没有执行 B 线程的时候取的值。这就导致，虽然两个线程都执行了这样一个操作，但是就结果而言，只算是执行了一次操作。其根本原因就是因为操作`x = y + 1`不是原子性的，而线程锁的作用，就是将一段操作变成“逻辑上”的原子性。线程锁的功能，就是使某段代码一次只能一个线程执行，即只有获得“锁”的线程才能进入，其他的线程只能等待获得“锁”。

Java 中有两类线程锁，一种是以 synchronized 关键字为核心的"自动型"线程锁，还有一种是以 ReentrantLock 为代表的"手动型"线程锁。二者在使用上的区别主要如下：

```java
private Object lock = new Object();
public void syncTest() {
    synchronized (lock) {
        try {
            lock.wait(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public void lockTest() {
    ReentrantLock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
    
    lock.lock();
    try {
        condition.await(1000, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}
```

由上可见一斑，synchronized 自动完成了上锁与解锁的操作，只需要使用这个关键字包含住需要上锁的代码即可，而 ReentrantLock 需要实例化一个对象，然后主动调用`lock()`和`unlock()`才行。下面就这两种做详细分析。

### synchronized

synchronized 有两种使用方式，**方法同步**和**代码块同步**，其使用方式如下：

```java
// 静态方法
public static synchronized fun() {}

// 成员方法
public synchronized fun() {}

// 对象
public fun() {
    synchronized (this) {}
}

// 类
public fun() {
    synchronized (Main.class) {}
}
```

上述每一种方式的限制略有区别，对于静态方法和类而言，它们锁住的是当前的类，所以当有多个对象调用这个方法的时候，只要它们是这同一个类的对象，那么它们就受到锁的限制，而成员方法和对象这种形式锁住的是某个具体的对象，所以对于同一个对象才会有锁的限制，不同的对象（即便是同一类）则没有限制。

因为 synchronized 只是一个修饰符，所以相关的锁操作是在 JVM 中实现的，依赖于对象的 monitor （监视器）。在 JVM 中每一个对象都对应着一个 monitor ，monitor 只能同时被一个线程获取，所以能够用于完成线程同步。比如，JVM 在处理 synchronized 代码块的时候，在代码块前面生成 monitorenter ，在代码块后面或者是异常捕获里生成 monitorexit 字节码，来实现的同步，在处理 synchronized 修饰的方法的时候，会给方法加上 ACC_SYNCHRONIZED 标识。而在一个线程要进入 synchronized 代码块的时候，首先要获取对应的 monitor ，获取到才会继续执行，否则就会进入阻塞状态。

也就是说，在使用 synchronized 的时候，实际上会在 JVM 中进行类似于 ReentrantLock 的处理，不过实现上不同。

在控制线程同步的时候光有锁还不行，还要有对应的同步操作，与 synchronized 配套的就是 Object 的`wait()`和`notify()`方法，它们只能在拥有锁的线程中调用，`wait()`方法调用之后线程会进入阻塞状态并释放锁，然后等待`notify()`的唤醒，`notify()`方法会唤醒调用了这个对象`wait()`方法的线程，将其由阻塞变成就绪状态。

并且，对于 synchronized 来说，也只能使用这个对象的`notify()`和`wait()`方法完成同步操作，这有一定的局限。如生产者-消费者模式的一个简单实现：

```java
public class Main {
    public static void main(String[] args) {
        Person person = new Person();

        ConsumerThread c1 = new ConsumerThread(person, "c1");
        ConsumerThread c2 = new ConsumerThread(person, "c2");
        ConsumerThread c3 = new ConsumerThread(person, "c3");
        ProducerThread p1 = new ProducerThread(person, "p1");
        ProducerThread p2 = new ProducerThread(person, "p2");
        ProducerThread p3 = new ProducerThread(person, "p3");

        c1.start();
        c2.start();
        c3.start();
        p1.start();
        p2.start();
        p3.start();
    }
}

public class Person {
    private final Object lock = new Object();
    private volatile int count = 0;

    public void consume() throws InterruptedException {
        synchronized (lock) {
            while (count <= 0) {
                lock.wait();
            }

            System.out.println("consume " + Thread.currentThread().getName());
            --count;
            lock.notifyAll();
        }
    }

    public void produce() throws InterruptedException {
        synchronized (lock) {
            while (count >= 2) {
                lock.wait();
            }

            System.out.println("produce " + Thread.currentThread().getName());
            ++count;
            lock.notifyAll();
        }
    }
}

public class ConsumerThread extends Thread {
    private Person person;

    ConsumerThread(Person person, String name) {
        super(name);
        this.person = person;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(500);
                person.consume();
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}

public class ProducerThread extends Thread {
    private Person person;

    ProducerThread(Person person, String name) {
        super(name);
        this.person = person;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(500);
                person.produce();
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}
```

在 Person 中，因为所有的生产者和消费者在执行的时候都需要同步，所以共用了一个锁 lock ，但是这在唤醒的时候是有问题的，因为共用了一个锁，那么执行完了一个生产操作，要唤醒一个消费者线程的时候，是应该调用`notify()`还是`notifyAll()`？事实上调用哪个都有问题。当调用`notify()`时，只会唤醒一个线程，但是此时等待锁的线程不仅有消费者，还有生产者，如果唤醒了其中的生产者线程，且容量已满不能再生产，那么生产者线程就会阻塞，此时会陷入死锁。当调用`notifyAll()`时，会唤醒所有的等待线程竞争 CPU ，如果还是生产者线程获得了锁，依旧会陷入死锁。

所以通常使用 synchronized 完成的生产者-消费者模型中，首先都会有一个 while 循环，判断是否满足条件，如在消费者中，当没有产品时就再调用`wait()`进入阻塞：

```java
while (count <= 0) {
    lock.wait();
}
```

### ReentrantLock

ReentrantLock 是 Java 中提供的另一种锁机制，与 synchronized 不同，ReentrantLock 是由 Java 内部实现的，在使用的时候需要调用`lock()`和`unlonk()`以上锁和释放锁，正因如此，ReentrantLock 一方面使用比较麻烦，另一方面也给开发者提供了更加灵活的功能。基本使用为：

```java
public void fun() {
    ReentrantLock lock = new ReentrantLock();
    lock.lock();
    // do something
    lock.unlock();
}
```

下面分析源码，看 ReentrantLock 的功能是如何实现的。

#### 构造函数

ReentrantLock 有两个重载的构造函数：

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

分别用于公平锁和非公平锁，可以看到，两种锁的区别就在于 sync 是 NonfairSync 还是 FairSync 实例。

#### 上锁

```java
public void lock() {
    sync.lock();
}

// FairSync
final void lock() {
    acquire(1);
}

// NonfairSync
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

FairSync 和 NonfairSync 都继承于 AbstractQueuedSynchronizer ，它们对`lock()`方法的不同实现就是它们公平与不公平的原因，首先看公平锁，这个方法会调用`acquire()`。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

这里调用了两个方法判断是否可以获得锁，首先调用`tryAcquire()`，这是一个非阻塞方法，判断是否可以获得锁，可以就获得，返回 true ，不可以就返回 false 。这个方法在 AbstractQueuedSynchronizer 没有实现，而是其子类 FairSync 和 NonfairSync 有具体的实现，也正是这个方法的不同实现，导致了其公平性的差异。首先看 FairSync 中的实现：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

首先获取 state 值，当其为 0 时意味着目前还没有人获得锁，则自己可以参与竞争，然后紧接着调用了`hasQueuedPredecessors()`方法，这个方法用于判断在当前线程之前是否还有别的线程正在等待，按照公平锁的规则，如果有别的线程比自己先请求，则自己不能获取锁，`hasQueuedPredecessors()`返回 false 则前面没有别的线程，然后使用 CAS 改变 state 值，修改成功则表明获取锁成功。

除此之外，还调用了`setExclusiveOwnerThread()`方法，并且当 state 不等于 0 （已经有线程获取锁的时候），没有直接返回 false ，而是判断了`current == getExclusiveOwnerThread()`，这是何意？其实这就是“重入锁”的实现，保存获取锁的线程，然后在已经获得锁的线程中再次调用获取锁，也能够获得，如下面这种方式：

```java
ReentrantLock lock = new ReentrantLock();
public void fun1() {
    lock.lock();
    fun2();
    lock.unlock();
}

public void fun2() {
    lock.lock();
    // do something
    lock.unlock();
}
```

因为 ReentrantLock 是一个重入锁，所以上面的代码可以正常执行。

上面说到这个方法的实现是公平锁和非公平锁的区别所在，再看 NonfairSync 的实现：

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这与上面的实现几乎相同，唯一不同的就是获取锁的时候，公平锁通过`hasQueuedPredecessors()`方法判断了前面是否有线程在等待，而非公平锁则没有这个判断，它会直接调用 CAS 尝试改变 state 值获取锁，所以，就会出现后到先获得锁的情况，是为不公平。

然后再回过头看`acquire()`方法，如果不同通过`tryAcquire()`获得锁，就会调用`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`，将当前线程加入等待队列等待。首先，调用了`addWaiter()`将当前线程封装成结点，实现如下：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}

Node(Node nextWaiter) {
    this.nextWaiter = nextWaiter;
    U.putObject(this, THREAD, Thread.currentThread());
}

private final void initializeSyncQueue() {
    Node h;
    if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
        tail = h;
}
```

首先创建一个 Node 实例，然后将其加入到当前队列尾部，如果队列为空还会先初始化队列。

然后看`acquireQueued()`方法：

```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

内部一个死循环，首先判断当前结点的前驱结点，如果前驱结点等于 head ，意味着当前结点是等待队列中的第一个结点，然后调用`tryAcquire()`获取锁，获取成功则将当前结点移出队列，并退出循环。否则就会接着往下执行，依此调用`shouldParkAfterFailedAcquire()`和`parkAndCheckIntercept()`。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

默认情况下结点的 waitStatus 是 0 ，所以按照上面的逻辑，第一次调用这个方法会将 waitStatu 置为 Node.SIGNAL ，第二次调用则会返回 true ，返回 true 意味着当前线程可以 park 了，则会调用`parkAndCheckInterrupt()`阻塞当前线程等待 unpark 。

至此，上锁的过程执行完毕，在这个过程中只有一个线程能够获得锁，冲破`lock()`的壁垒，继续往下执行，其他的线程都只能迷失在`acquireQueued()`中，等待唤醒。

#### 释放锁

一个线程执行完了之后，需要将自己的锁释放掉，其他线程才能继续执行，释放锁的方法为`unlock()`。

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

基本的逻辑如上，ReentrantLock 的`unlock()`方法会转而调用 AbstractQueuedSynchronizer 的`release()`方法，首先，判断当前线程是否是获得锁的线程，不是则抛出 IllegalMonitorStateException 异常。然后，设置 state 值，设置 exclusiveOwnerThread 以释放锁，最后返回是否释放成功。

然后回到`release()`，如果释放成功，则判断是否还有其他等待线程，其他等待线程是否在阻塞状态，如果是，则调用`unparkSuccessor()`方法，将后继结点 unpark 。

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

本方法的逻辑为，先找到一个在阻塞状态的结点（线程），然后调用`LockSupport.unpark()`是这个线程跳出 park 状态，继而走向上面讲的上锁过程中的`acquireQueued()`方法获取锁。

至此，释放锁的过程也结束了。

但是光有锁只能完成线程互斥，并不能完成线程间的同步。所以 ReentrantLock 还提供了与之配套的 Condition ，这就像是对象之于 synchronized ，它提供了与`wait()`和`notify()`相对应的`await()`和`signal()`方法。

#### Condition

与 synchronized 不同的是，ReentrantLock 可以有多个 Condition ，且 Condition 之间是独立的，这在实现生产者-模式中模型中显得更加友好。实现如下：

```java
public class Person {
    public static final int MAX = 2;

    private ReentrantLock lock;
    private Condition produce;
    private Condition consumer;
    private volatile int count;

    public LockPerson() {
        lock = new ReentrantLock();
        produce = lock.newCondition();
        consumer = lock.newCondition();
    }

    public void produce() throws InterruptedException {
        lock.lock();
        try {
            if (count > MAX) {
                produce.await();
            }

            ++count;
            System.out.println("produce " + Thread.currentThread().getName());
            consumer.signal();
        } finally {
            lock.unlock();
        }
    }

    public void consume() throws InterruptedException {
        lock.lock();
        try {
            if (count == 0) {
                consumer.await();
            }

            --count;
            System.out.println("consume " + Thread.currentThread().getName());
            produce.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

这就是使用 ReentrantLock 对生产者-消费者模型的重新实现，与上面相比其实也只需要改动 Person 类。这里只需要使用两个 Condition 变量将生产者和消费者区分开，就不需要使用 while 循环判断条件了。下面看看 Condition 的实现原理，以`await()`方法开始：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

final int fullyRelease(Node node) {
    try {
        int savedState = getState();
        if (release(savedState))
            return savedState;
        throw new IllegalMonitorStateException();
    } catch (Throwable t) {
        node.waitStatus = Node.CANCELLED;
        throw t;
    }
}
```

Condition 的实现类是 ConditionObject ，在这个类内部有一个链表，保存着所有调用了`await()`的线程，与 AbstractQueuedSynchronizer 类似，也是将线程包装成 Node 结点，在`await()`方法中，先调用`addConditionWaiter()`生成一个结点并将其加入等待链表，然后接着调用`fullyRelease()`释放掉当前线程锁（因为可能有重入锁的情况，这时 state 大于 1 ，但是释放锁需要将 state 置 0 ，所以叫 fully release）。然后有一个 while 循环，调用`isOnSyncQueue()`方法判断当前结点是否存在于同步队列（也就是 AQS 中的等待锁线程队列）。如果不在这个队列，就会一直调用`LockSupport.park()`阻塞当前线程。当前期间可能会因为中断等原因退出循环。

紧接着，当退出 while 循环之后，便会调用`acquireQueued()`方法，上面讲到，这个方法是线程用于获取锁的，那么此时调用这个方法是何用意？首先要说明 while 循环什么时候跳出，除了某些情况下因为中断跳出之外，一般情况下只有调用了`singal()`方法才会跳出这个循环（调用`singal()`方法时会将 Node 结点加入 AQS 的等待队列，也就不满足`!isOnSyncQueue()`条件了），所以，当调用了`singal()`之后，线程就会进入等待锁的行列，直到得到线程锁之后，才继续执行`await()`方法之后的操作。

下面再看`singal()`的实现：

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;
    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

private Node enq(Node node) {
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return oldTail;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

首先，从 Condition 的等待队列中取出第一个结点 first ，调用`doSingal()`方法，这个方法里是一个循环，不断调用`transferForSingal()`直到有一个操作成功。在`transferForSingal()`中，调用了`enq(node)`方法将这个结点加入了 AQS 的等待队列中，并调用`LockSupport.unpark()`唤醒这个结点对应的线程，这一步对应的就是`await()`方法中的 while 循环，当这个结点从 Condition 链表转移到 AQS 的队列中之后，就意味着调用了`singal()`方法，所以需要退出循环。

总的来说，Contion 内部有一个等待队列，所有调用了`await()`方法的线程，首先将线程封装成结点加入 Condition 的等待队列等待`singal()`唤醒，另一方面还会释放这个线程的锁，然后阻塞该线程。等到`singal()`调用之后，该线程被唤醒，并将 Node 结点移到 AQS 的等待队列，恢复原先的 state ，继续请求线程锁。

正是由于每一个 Condition 都有一个独立的等待队列，所以使得多个 Condition 的`await()`和`singal()`互不相关，其使用场景也更加灵活。

### 总结

上面两种锁在许多方面都有着不同之处，下列若干：

#### 可中断性

synchronized 是不可中断锁，当需要请求线程锁的时候，直接就是使用`synchronized(lock)`，线程会陷入阻塞直到获得线程锁，而 ReentrantLock 是一个可中断锁，最明显的就是`lockInterruptibly()`方法，这个方法会抛出异常，在请求锁的过程中，如果当前线程不想继续等待锁而是去做其他事情，就可以中断这个线程，然后捕获`lockInterruptibly()`抛出的异常执行其他工作。

#### 公平性

上面也提到了，ReentrantLock 在初始化的时候可以提供一个参数规定是否公平，上面也讲到了 ReentrantLock 公平性实现的原理，但是在 synchronized 中则没有公平这一概念，它默认就是一个非公平锁。

#### 多条件

在 ReentrantLock 中可以通过`newCondition()`绑定出多个条件，每个 Condition 之间的等待与唤醒操作是独立的，而在 synchronized 中只能使用一个对象的完成等待与唤醒操作，其区分在上面的生产者-消费者模型的实现中可以看出，在这方面 ReentrantLock 的使用在灵活性上更胜一筹。

#### 可重入性

重入锁指的是如果当前线程已经获得了锁，那么在当前线程再次请求获得锁的时候是可以成功的，上面也讲到了 ReentrantLock 重入锁的实现方法，synchronized 也是一个可重入锁。

#### 性能

目前来说，在线程数量不多的时候二者性能相差不多，当有大量线程竞争锁的时候还是 ReentrantLock 效率更高。在早些时候，Java 5 及其之前，synchronized 性能较低，不过在之后的版本中对其进行了一些优化，使得它的性能有较高提升，不会闭 ReentrantLock 差太多，所以在一般情况下，如果 synchronized 能够完成目标功能的情况下，应该优先使用它，毕竟使用起来很方便。