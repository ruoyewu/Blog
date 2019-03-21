---
title: Java Map
date: 2019-03-21 15:00
tags:
	- java
---

在 Java 中，Map 是与 List 一样的最常用的数据结构之一，相比于 List ，Map 是一种键-值存取的数据结构，也是一种高效的查找数据方案，相较于 List 的遍历查找，Map 可以使用诸如哈希表、二叉搜索树的方式查找。Map 是一个接口，其实现主要有四个类：

1.  HashMap
2.  LinkedHashMap
3.  TreeMap
4.  Hashtable
5.  WeakHashMap
6.  ConcurrentHashMap

每个类都实现了 Map 接口，但是各类的具体实现、使用场合等略有不同。除了 Hashtable 之外，其他五个类都继承于 AbstractMap 这个抽象类。下面一一介绍：

### AbstractMap

AbstractMap 实现了 Map 接口，其本身并没有关于数据存储的实现，但是却有诸如`size()`、`get()`方法的实现，因为它用到了 Iterator ，与 AbstractList 一样，通过抽象方法`entrySet()`得到 EntrySet ，然后再利用 EntrySet 实现一些方法。

### HashMap

HashMap 继承于 AbstractMap ，其内部使用哈希表做数据存储。常用的哈希表冲突处理有两种方式：拉链法和线性探测法。线性探测法是指当某一个位置出现了冲突时，利用一个函数计算出下一个可以存放当前数值的地址，再检测下一个地址是否可以存放，使用这种方法的时候，所有的值都是直接存放在哈希表中的，当哈希表满（或者快满）的时候，就需要扩大哈希表，否则肯定会出现大量的冲突。而拉链法则没有此顾虑，因为拉链法处理冲突的方式是将所有映射到某一地址的数据都放到一起，使用某种数据结构（如链表）将其联系起来，哈希表中只存放链表的指针，当索引到某一地址之后，继而查询这个链表，所以理论上不会出现无空间可放的情况。但是拉链法导致的问题就是，如果有太多的冲突，哈希表的优势就荡然无存了。

在 Java 的 HashMap 中使用的是拉链法，但是不同于一般的拉链法，这里的「链」用的不光是是链表，还可以是红黑树，红黑树作为一种二叉查找树，能够利用其优势进一步提升 HashMap 的查询效率。在 HashMap 内部就同时使用了这两种数据结构作拉链，当某一个链表的长度大于某个定值（一般为 8）的时候，这个链表就会被转化为红黑树进行存储。

从它的`put()`方法可见一斑：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

每当需要插入一个键值对时，先根据 key 的 hash 值索引到对应的位置，有三个判断：

1.  当前位置为 null，创建一个 Node 并放在这个位置
2.  当前位置为 Node，当前采用链表存储，向链表插入结点，并在插入之后判断长度
3.  当前位置为 TreeNode，当前采用红黑树存储，向红黑树插入结点

分支2中，当`binCount >= TREEIFY_THRESHOLD-1`时，就会调用`treeifyBin()`方法，将这个链表转化为红黑树。

### TreeMap

与 HashMap 相比，TreeMap 相当于是一个哈希表长度始终为 1 的 HashMap ，TreeMap 同样是采用红黑树作为数据存储结构，说是 HashMap 的一种特例也不为过，不过它们还是有些不同：在 HashMap 中，红黑树的结点之间的大小判断是利用 key 的 hash 值，但是在 TreeMap 中并没有使用这样一种方式，而是提供了一种更为灵活的方式 Comparator 或 Comparable ，通过传入一个 Comparator 对象或者使 key 继承 Comparable 使 key 之间具有可比性，在存储的时候按照这种顺序进行存储，所以 TreeMap 还额外提供了诸如`firstEntry()`、`lastEntry()`等方法。以`put()`方法为例：

```java
public V put(K key, V value) {
    TreeMapEntry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        root = new TreeMapEntry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    TreeMapEntry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    TreeMapEntry<K,V> e = new TreeMapEntry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

这个方法有两个分支，对应着 Comparator 是否为 null ，当其不为 null 的时候，利用这个类进行 key 之间的比较，当其为 null 的时候，就会将 key 强制转换成 Comparable ，插入完成之后再调用`fixAfterInsertion()`方法进行红黑树调整。总的来说，相对于 HashMap 的数据「高效的存储」，TreeMap 给我的感觉是更注重于数据「有序的存储」，所以就有了 Comparator ，所以就有了红黑树。

### LinkedHashMap

LinkedHashMap 继承于 HashMap ，意味着他们的基本功能是一致的，但是多了一个「Linked」，正如 TreeMap 的出现是为了解决 HashMap 不能有效地存储有序数据一样，LinkedHashMap 显然也是为了解决 HashMap 的「不连续性」而存在的，不过不是采用替换存储结构的方式，而是在原本的基础上增加一个链表，将 HashMap 中所有的数据按照某种顺序排列。

LinkedHashMap 相比于 HashMap ，多了两个特性，有序性和有限性。

对于有序性，LinkedHashMap 能够将所有 HashMap 中存储的数据保存在一个链表中，这就能够使得整个哈希表中的数据能够在一起比较先后。其实现方法主要是 LinkedHashMap 中重写的`newNode()`和`newTreeNode()`方法。

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMapEntry<K,V> p =
        new LinkedHashMapEntry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
    LinkedHashMapEntry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

如上的`newNode()`方法，在 HashMap 中所有的新增的 Node 都需要使用`newNode()`方法生成，而 LinkedHashMap 重写了这个方法，增加了`linkNodeLast()`，将其加入到自己持有的 LinkedHashMapEntry 链表的尾部，即所有需要添加到 HashMap 中的结点都首先被添加到了 LinkedHashMap 的链表中。另外，链表中的顺序有两个，一个是插入顺序，即按照插入先后排序，另一种是访问顺序，看`get()`方法：

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

需要访问一个结点的时候，会根据 accessOrder 的值选择是否调用`afterNodeAccess()`方法，这个方法会将当前访问的结点移动到链表的尾部，accessOrder 的值是在构造实例的时候确定的。

对于有限性，看方法`afterNodeInsertion()`：

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMapEntry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

在插入数据之后，都会调用`afterNodeInsertion()`方法，在这个方法里又会调用`removeEldestEntry()`方法判断是否要移除这个结点，开发者可以继承 LinkedHashMap 并重写这个方法，根据自己的需要决定是否要删除这个 eldest 结点（前面说到有序性，可以根据插入顺序和访问顺序对所有结点排序，所以表头结点就是那个最先插入或者是最不经常访问的结点），如可以这么重写：

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > MAX_LENGTH;
}
```

当当前的结点数量大于最大长度 MAX_LENGTH 的时候，就移除 first 结点。Android 中的 LruCache 就是使用 LinkedHashMap 实现的。

### Hashtable

与前面三个不同，Hashtable 并没有继承 AbstractMap ，与 HashMap 相同，Hashtable 内部也使用哈希表存储数据，使用拉链法处理冲突，不过不同的是，没有红黑树的加持，Hashtable 之于 HashMap ，类似于 Vector 之于 ArrayList ，采用相同的数据结构存储，但是做到了线程安全：几乎所有的方法都使用 synchronized 修饰。除了这一点之外，好像就没什么可再说的了。

### WeakHashMap

看到 WeakHashMap ，就能想到是与 WeakReference 相关的，确实，对于一个 HashMap 来说，所有保存在这里面的值都不能被释放掉，因为 HashMap 对它们有着强引用，所以就有可能造成大量的内存浪费。开发过程中很可能有一种需求，只需要使用 HashMap 进行一些数据查询操作，但是又不希望所有放在 HashMap 中的数据不能被释放，即在外部不会再用到这个对象的时候，GC 能够将这个对象回收。

所以就有了 WeakHashMap 的出现，它基本上也是使用哈希表保存数据，但并不是直接保存 key 的对象，而是保存 key 的弱引用，当外界不存在对这个 key 的强引用或软引用的时候，GC 就可以将这个 key 回收。另外，哈希表保存的是键值对，但是 WeakHashMap 只保存了 key 的弱引用，value 还是使用强引用保存的，所以 WeakHashMap 的做法是，当 key 被回收的时候，就主动释放掉对 value 的引用。

首先，WeakHashMap 的 Entry 继承了 WeakReference ，由此完成弱引用，其构造方法为：

```java
Entry(Object key, V value,
      ReferenceQueue<Object> queue,
      int hash, Entry<K,V> next) {
    super(key, queue);
    this.value = value;
    this.hash  = hash;
    this.next  = next;
}
```

`super(key, queue)`调用的就是 WeakReference 的构造方法，将 key 作为弱引用传入，然后 value 仍被强引用着。

然后，每当需要对哈希表操作的时候，都会调用`getTable()`方法移除那些已经被释放的 key 对应的结点：

```java
private Entry<K,V>[] getTable() {
    expungeStaleEntries();
    return table;
}

private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);
            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

`expungeStaleEntries()`方法遍历了所有被释放的 key 对应的 Entry 对象（也就是 WeakReference 对象，它继承了 WeakReference），将其从哈希表中移除，并将这个 Entry 对象对 value 的引用，使得 value 也可以被回收。可以看到，在 WeakHashMap 中，key 与 value 并不是同时被回收的，key 一般是在外部没有了它的引用时由 GC 自动回收，而 value 是在回收了 key 之后，调用`getTable()`时，通过检测已回收的 key 然后主动释放掉 value 的引用，以等待下次 GC 回收。

如在`put()`方法中就有对`getTable()`的调用：

````java
public V put(K key, V value) {
    Object k = maskNull(key);
    int h = hash(k);
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length);
    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        if (h == e.hash && eq(k, e.get())) {
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue;
        }
    }
    modCount++;
    Entry<K,V> e = tab[i];
    tab[i] = new Entry<>(k, value, queue, h, e);
    if (++size >= threshold)
        resize(tab.length * 2);
    return null;
}
````

### ConcurrentHashMap

这是`java.util.concurrent`包下的一个类，毫无疑问，它是用于多线程操作的。就其功能而言与 Hashtable 是差不多的，但是不同于 Hashtable 大量的 synchronized 的使用，ConcurrentHashMap 使用更为精细的方法，如大量 volatile、Unsafe 的使用，还是用了与 HashTable 一样的链表与红黑树共同使用的方式，相比于 Hashtable 而言效率会更高，更适合用于并发编程环境。不仅如此，在这个包下还有其他一些并发编程适用的数据结构，如 ArrayBlockingQueue、CopyOnWriteArrayList 等。

对一个 HashMap 而言，最耗时的操作莫过于重建哈希表了，这一步骤需要申请内存、重新计算 hash 值和覆盖原哈希表这些过程，而 ConcurrentHashMap 可以使用多线程进行表的重建，具体的实现是`transfer()`方法，大致原理如下：

在哈希表中，每个数据都会通过 hash 值索引到对应的位置，每个位置可是是一个链表（或二叉树）保存着所有索引到当前位置的结点，那么重建哈希表的过程，就是原本的每一个链表拆分到新的哈希表中，如果哈希表的长度为 n ，当前地址为 i ，重建的表长度为 2n ，那么位置 i 处的链表经过拆分之后会分成两个链表，索引位置分别为 i 和 i+n 。也就是说，每一个链表的拆分过程是独立的，链表之间的拆分并不互相干涉，因为它们拆分之后的位置不会出现重合。这就给并发操作提供了条件，可以使用多个线程共同完成新建哈希表，每个线程完成一个链表的拆分，当所有链表都拆分完成之后，新的哈希表就构建成功了。多个线程共同拆分一张哈希表，但是对于每一个链表而言，还要加上线程锁，保证每个链表的拆分只由一个线程完成。

所以这就导致在 ConcurrentHashMap 的 table 中有三类结点：

1.  链表结点，Node
2.  二叉树结点，TreeNode
3.  正在拆分结点，ForwardingNode

前两种结点的属于正常存储方式，与 HashMap 中的功能一致，而第三种，就是指某个线程正在执行这个链表的拆分工作还不能被操作，此时会调用`heloTransfer()`方法让当前这个线程也加入到重建表的行列中去，直到重建完成，再继续其操作。

### Summary

上面就 Java 中提供的一些可能会常用的 Map 数据类型给出了一些介绍，究其根本，它们都是实现了 Map 接口，基本的数据存储功能都能得到保障，但是光是能够完成某项功能还不够，还要能够针对不同的环境、不同的特点，更出色地完成这个功能，所以衍生出了这么多 AbstractMap 的子类。作为一个开发者也是，不能仅以完成某个任务为目标，真正应该思考的是如何更好地完成某项任务。