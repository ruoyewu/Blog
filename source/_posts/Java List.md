---
title: Java List
date: 2019-02-13 23:00
tags:
	- java
---

在 Java 中虽然有数组这个东西，但是保存一组数据的时候最常用的还是 List ，因为在使用数据的时候数据的长度往往是不固定的，所以要在使用之前先声明长度、并且在声明之后长度便不可变的数组，显然不是一个好的数据结构。

List 有许多实现类，常用的大致两种，ArrayList 和 LinkedList ，分别是直接存取和顺序存取，就像是数组与链表的区别，不过也不太一样。

### ArrayList 直接存取

在数据结构中要做到能够直接存取，只有使用数组，所以 ArrayList 实现这个功能也用到了数组，说白了，就是 ArrayList 内部维护了一个可变长数组，在存取的时候，ArrayList 先负责一些关于安全性的判断，如是否越界等，然后再操作这个数组，或者也可以说 ArrayList 是数组的一种功能拓展性包装，不过既然功能有了拓展，就避免不了性能上的缺失。

首先说 ArrayList 的初始化，在创建一个 ArrayList 对象的时候，它会自动生成一个数组，数组长度默认为 0 。

```java
/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

然后添加元素的时候，ArrayList 会先判断数组是否已满。

```java
/**
 * This helper method split out from add(E) to keep method
 * bytecode size under 35 (the -XX:MaxInlineSize default value),
 * which helps when add(E) is called in a C1-compiled loop.
 */
private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}
```

如果数组已满，就需要调用`grow()`方法对数组扩容，大致就是创建一个新的、容量更大的数组，将已有的数组取而代之。我觉得 ArrayList 相比于数组，最大的弊端就在于此，随着不断增加元素，要不断的创建新的数组以容纳更多的元素。不过为了兼具直接存取与变长存储，这些损失也算是值得的。

然后，删除元素的时候，先找到删除元素的位置，再利用数组的复制功能，将这个位置之后的所有元素全部前移一位。

```java
/**
 * Removes the element at the specified position in this list.
 * Shifts any subsequent elements to the left (subtracts one from their
 * indices).
 *
 * @param index the index of the element to be removed
 * @return the element that was removed from the list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E remove(int index) {
    Objects.checkIndex(index, size);
    final Object[] es = elementData;
    @SuppressWarnings("unchecked") E oldValue = (E) es[index];
    fastRemove(es, index);
    return oldValue;
}

/**
 * Private remove method that skips bounds checking and does not
 * return the value removed.
 */
private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize;
    if ((newSize = size - 1) > i)
        System.arraycopy(es, i + 1, es, i, newSize - i);
    es[size = newSize] = null;
}
```

删除也算是挺麻烦的，要先找到位置，然后移动整个数组。

另外，ArrayList 中保存数据的数组是一个`Object[]`。我曾经也自己写过类似的 ArrayList ，不过在使用数组的时候首先想到的是`T[]`，但是在后来的使用中发现，并不能直接使用`T[] elements = new T[length]`，而这里的`Object[]`就很好的解决了这个问题，所有的类默认都是 Object 的子类，那么无论之后的 T 是什么，在存放的时候都可以直接存放，然后取出的时候可以使用强制类型转换变成 T 所代表的类型，这算是看源码学到的一个小知识。

### LinkedList 顺序存取

LinkedList 就是一个货真价实的双链表，结点定义如下：

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

添加删除什么的也就是操作结点。另外有一点需要注意，那就是 LinkedList 还是 Deque 的实现，也就是说 LinkedList 还有一个身份：双端队列。除此之外，LinkedList 的特性就是链表的特性：在频繁增删的操作中效率较高，但是在存取较多的操作中还是不如 ArrayList 。

### Vector 线程安全

实现 List 接口的类还有一个 Vector 比较特殊，它与 ArrayList 一样，都是通过维护一个可变数组实现功能，细看的话，各个功能的实现也是相差不大。唯一的区别可能就是 Vector 在许多方法上都加了 synchronized 关键字，通过这个关键字也可以看出，Vector 应该就是为了解决多线程时的安全问题而单独增加的一个线程安全列表。

比如`add(E)`方法：

```java
/**
 * Appends the specified element to the end of this Vector.
 *
 * @param e element to be appended to this Vector
 * @return {@code true} (as specified by {@link Collection#add})
 * @since 1.2
 */
public synchronized boolean add(E e) {
    modCount++;
    add(e, elementData, elementCount);
    return true;
}
```

与 ArrayList 的`add(E)`方法就差了一个`synchronized`关键字，其它的很多方法也是如此。

### AbstractList 骨骼

上面的 ArrayList 、Vector 、LinkedList 都是 AbstractList 直接或间接的子类，那么 AbstractList 存在的意义是什么呢？一般来说，多个子类继承同一个父类，那么原因当然就是这几个子类中有一些相同的操作，为了使它们的相同的功能不重复编写，只需继承同一个实现了这些方法的父类即可。

如果要具体了解一个类意义，自然要从这个类所含有的成员变量和方法出发。AbstractList 只有一个成员变量，`modCount`，这个变量会在 List 的长度发生改变时变化，在 Iterator 中使用，用来判断`ConcurrentModificationException`这个异常，大概就是说当有同一个 List 的多个 Iterator 在工作的时候，不能在任何一个 Iterator 中执行修改 List 长度的操作（如 add、remove），如下会报错：

```java
public class Main {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5, 6, 7));
        Iterator<Integer> i1 = list.iterator();
        Iterator<Integer> i2 = list.iterator();
        System.out.println(i2.next());
        System.out.println(i1.next());
        i2.remove();
        System.out.println(i2.next());
        System.out.println(i1.next());
    }
}
```

另外，AbstractList 中的方法大致包括三类：

1.  虽然有实现，但是是个空实现，直接调用会抛出异常，如`add()`、`remove()`，至于为什么使用一个不能用的方法而不是使用一个抽象方法，我也不清楚。
2.  有实现，但是依赖 Iterator ，在子类中往往会重新根据存储结构重新实现，但不是必须要重新实现的，只要子类中实现了`listIterator()`方法，这些方法都能够正常工作，如`clear()`
3.  有实现，并且子类会沿用此方法，也依赖于 Iterator ，如`equal()`、`hashCode()`

总而言之，Abstract 搭建起了一个 List 的骨干框架，实现了部分 List 的功能。