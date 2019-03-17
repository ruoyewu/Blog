---
title: 设计模式 - 单例模式
date: 2018-05-03
tags: 
	- design pattern
---

## 介绍

单例模式，即一个类只有一个实例，所有要使用这个类的时候，使用的都是这同一个实例，基本做法就是，在类的内部存放一个静态的实例，然后类再提供一个静态方法以供其他类获取到这个实例。如：

```java
public class Singleton {
    private static Singleton sInstance;
    
    private Singleton () {
        // init
    }
    
    public static getInstance() {
        return sInstence;
    }
}
```

单例模式最重要的问题，就是如何得到这个实例，以及在什么时候得到这个实例。由此就得到了数种不同的单例模式。

## 种类

### 饿汉式

```java
public class Singleton {
    private static final Singleton sInstance = new Singleton();
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        return sInstance;
    }
}
```

饿汉模式，使用`static final`标记实例，在类加载的时候就进行初始化，所以是线程安全的，但是缺点有二：

1. 可能在本次程序运行中并没有使用到这个单例，但它还是被初始化了
2. 如果这个单例构造的时候需要一些其他的参数，而那些参数是在程序运行中产生的，就不能使用这种方法

### 懒汉式

```java
public class Singleton {
    private static Singleton sInstance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (sInstance == null) {
            sInstance = new Singleton();
        }
        return sInstance;
    }
}
```

这种方法解决了上述的饿汉模式的不足，仅在第一次调用`getInstance()`方法的时候才进行实例的初始化，但是很显然，这种方法不是线程安全的，于是有了这种改进：

```java
public class Singleton {
    private static Singleton sInstance;
    
    private Singleton() {}
    
    public static synchronized Singleton getInstance() {
        if (sInstance == null) {
            sInstance = new Singleton();
        }
        return sInstance;
    }
}
```

这样的懒汉模式确实解决了线程安全的问题，但是锁的过度使用却造成了执行效率的降低，本来只需要在第一次初始化的时候加锁，而现在每一次获取实例都要加锁，于是又有了后面的只在第一次初始化加锁的双重检验锁方法。

### 双重检验锁

```java
public class Singleton {
    private static Singleton sInstance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (sInstance == null) {
            synchronized (Singleton.class) {
                if (sInstance == null) {
                    sInstance = new Singleton();
                }
            }
        }
        return sInstance;
    }
}
```

这种方法会首先判断一次，当 sInstance 为空的时候，再在加锁的环境下进行初始化，不过它还是有问题，`sInstance = new Singleton()`在 JVM 中执行实际经历了三个阶段：

1. 给 sInstance 分配内存
2. 调用 Singleton 的构造函数执行初始化
3. 将 sInstance 指向 Singleton 实例所在地内存空间（此时 sInstance 不为 null ）

不过在 JVM 执行的过程中，这三个步骤因为指令重排序的原因，可能会变成 1 3 2 的执行步骤，导致其他线程调用`getInstance()`方法的时候，即使初始化还没完成，也会返回 sInstance 指向的对象（未完成初始化的对象），导致错误。有一个解决方法就是使用 volitile 关键字：

```java
public class Singleton {
    private volatile static Singleton sInstance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (sInstance == null) {
            synchronized (Singleton.class) {
                if (sInstance == null) {
                    sInstance = new Singleton();
                }
            }
        }
        return sInstance;
    }
}
```

使用 volitile 关键字的主要原因，就是 volitile 禁止了 JVM 的指令重排序，但是这样会一定程度降低了 JVM 的执行效率。

### 静态内部类

```java
public class Singleton {
    private Singleton() {}
    
    public static Singleton getInstance() {
        return SingletonHolder.sInstance;
    }
    
    private static class SingletonHolder {
        private static final Singleton sInstance = new Singleton();
    }
}
```

这种方法使用了静态内部类，将实例保存在静态内部类里面，然后调用`getInstance()`方法从内部类里面取得实例，并且由于内部类的关系，实例只会在调用`getInsatnce()`方法的时候初始化，同时它是线程安全的，所以，总的来说，这是一种不错的实现单例模式的方法。

### 枚举

```java
public enum Singleton {
    INSTNCE;
    
    // some methods
}
```

实现单例还能使用枚举完成，如上所示，使枚举拥有一个实例 INSTANCE ，然后像使用普通的类一样添加方法、添加成员变量等等。因为枚举本身的特性，使得它是线程安全的。

### 容器实现

```java
public class Singleton {
    private static Map<String, Object> objMap = new HashMap<>();
    
    private Singleton() {}
    
    public static void registerService(String key, String instance) {
        if (!objMap.containsKey(key)) {
            objMap.put(key, instance);
        }
    }
    
    public static Object getService(String key) {
        return objMap.get(key);
    }
}
```

这种方式并不能阻止类的多次实例化，而是通过使用者有意识的使用下，在最初的时候将一个实例保存到容器里面，然后需要实例的时候从容器中拿出，这样每次获得到的肯定也就是同一个实例了，不过这种方式一般多用于一系列单例的管理。

## 使用

总的来说就是，当一个类能够满足同一个实例可以在多个地方重复使用的时候，就可以使用单例模式实现实例的复用，降低创建对象的消耗。