---
title: Java ThreadLocal 原理
date: 2019-04-02 19:00
tags:
	- java
---

在操作系统中，每个进程都有自己的内存空间，而所有的线程共享同一片内存，但是有些时候开发者可能希望保存一些线程内部的数据，不与其他线程共享。如 Android 中的消息机制中的 Looper ，其对象就是一个线程私有的，每一个 Looper 对象对应着一个线程。这种需求在内存的角度是不能实现的，只能从可达性上保证其他线程不能访问。Java 中的 ThreadLocal 就是这样一种实现。

在 Java 中每个线程对应着一个 Thread 对象，所以所谓的线程内私有变量 ThreadLocal ，其实就是作为这个 Thread 成员变量存在的，正因为操作系统中运行的线程成为了 Java 中逻辑上的 Thread 对象，所以，操作系统中不支持的线程私有，可以由 Java 中对象的封装特性使其换一种方式表现出来，因为每一个 Thread 对象之间的成员变量是独立的。

ThreadLocal 的原理就如上所述，下面看其实现过程，以`set()`方法为例：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

实现还是挺简单的，先通过`Thread.currentThread()`得到当前的线程，然后得到其 ThreadLocalMap 对象，ThreadLocalMap 就是一个哈希表，不同于 HashMap ，它的冲突解决方案是线性探测。

```java
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

于是便将变量存到了 Thread 的成员变量中成为 Thread 的私有数据，再看取数据`get()`方法：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

可以看到，在`get()`方法内部也是先通过`Thread.currentThread()`得到当前线程实例，再取其 ThreadLocalMap 查找，但是就`get()`方法而言，它是一个无参数的方法，并且 ThreadLocal 也是一个线程共享的变量，但是调用`get()`方法却可以根据当前所处线程得到不同的数据，实现了”表面上”的线程私有。

为了解决 Map 存储内存泄漏的问题，ThreadLocalMap 也使用了与 WeakHashMap 相似的解决办法，结点 ThreadLocalMap.Entry 继承于WeakReference ，使用弱引用存储 key ：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

然后在查询数据的时候，如果发现某个位置的 key 为 null ，就意味着这里的 key 已经被回收了，此时会将 value 也释放掉。查询结点是`getEntry()`和`getEntryAfterMiss()`方法，回收方法为`expungeStaleEntry()`。其调用顺序为：当查找一个数据的时候，首先调用`getEntry()`查找对应位置是否正好是要查找的，如果不是，则意味着出现了冲突，此时调用`getEntryAfterMiss()`，在这个方法里循环线性探测，直到找到正确的数据为止。并且，在这个过程中，如果发现某个位置的 key 已经被回收了，则调用`expungStaleEntry()`方法将对应位置的 value 释放，并重新散列。