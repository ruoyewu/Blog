---
title: Java 内部类
date: 2019-03-03 17:00
tags:
	- java
---

## Java 内部类

Java 是一门面向对象的语言，所有功能的实现都是基于类的，类于类之间可以发生关系，如继承、实现接口，其中有一种关系叫做内部类，顾名思义，在某一个类内部定义的类叫做内部类。而内部类又可以根据定义的方式、位置、特点等，划分为四种：成员内部类、方法内部类、静态内部类和匿名内部类。

这四种内部类有一些相通之处，也有不同之处。

### 成员内部类

成员内部类，就是在一个类的内部，与成员变量、成员方法这些定义在同一个位置的内部类，它可以看作是一个类的成员，就如同成员方法能够无限制访问一个类的成员变量一样，内部类也可以无限制地访问它的外部类的所有成员方法和成员变量，而不需要考虑这些方法或者变量的限定修饰符。如下例子：

```java
public class Outer {
    private String str;

    private void outerDisplay() {
        System.out.println(str);
    }

    public Inner getInner() {
        return new Inner();
    }

    public class Inner {

        void innerDisplay() {
            System.out.println(str);
            outerDisplay();
        }
    }
}
```

使用成员内部类的时候一般有两种限制：

1.  只有先创建了外部类的对象，才能创建内部类的对象
2.  内部类中不能定义静态的方法或者变量

上面说到在内部类中可以无限制地访问外部类的方法和变量，但是毕竟它们是不同的类，没有 Java 中常见的那种继承机制，那么这种直接访问是如何实现的？原因就在于一个类的成员内部类会自动持有一个外部类的引用，在调用外部类的方法的时候，虽然在代码上看上去是直接调用的，但是通过 Java 编译器编译之后得到的 class 文件就可以看到，这其实还是通过外部类的引用得到的，如上述 Inner 这个类的代码：

```java
public class Inner {
    public Inner() {
    }
    public void innerDisplay() {
        System.out.println(Outer.this.str);
        Outer.this.outerDisplay();
    }
}
```

更加完整的形式是这样的，同时这也说明了为什么内部类的实例化需要依赖于外部类的实例：如果不先确定外部类的实例确实存在的话，那内部类的对于外部类的引用就无从得来了。

另一个问题，为什么成员内部类中不能使用静态方法和变量？我们知道，一个类的静态方法和变量，可以不通过实例化这个类直接访问，这是一般类的做法，我们在调用类的静态方法的时候 JVM 会自动加载这个类，但是对于内部类而言，则略有不同，对于成员内部类，虽然它是一个类，但它也是一个外部类的「成员」内部类，我们知道，对于一个外部类来说，它的类在加载的时候会随之加载它的静态方法/变量，但是它的成员方法/变量只有在这个外部类被实例化的时候才会加载，成员内部类也是这样。所以如果在某种情况下，没有先创建内部类的实例而去调用这个内部类的静态方法的话，因为类此时并没有加载，所以肯定会导致错误。

关于实例化方面，下面的例子就很好的说明了成员内部类依赖于外部类的实例存在。

```java
public class Main {
    public static void main(String[] args) {
        Outer outer = new Outer();
        Outer.Inner inner = outer.new Inner();
        inner.innerDisplay();
    }
}
```

### 方法内部类

方法内部类一般被定义在一个方法的内部或者是某一个作用域里面，此时这个内部类只在当前的域里能够被访问，所以具有比较好的隐蔽性，一般情况下使用方法内部类多是用来辅助解决某个问题，而这个类又不需要被外界知道的情况，如下面这个例子：

```java
public class Outer {
    interface People {
        void eat();
    }

    public People fun() {
        class Man implements People {

            @Override
            public void eat() {
                System.out.println("man eat");
            }
        }
        return new Man();
    }
}
```

因为是定义在一个方法里面的，所以这个类的加载时间与方法的加载时间保持一致：如果这是一个静态方法，会在外部类加载的时候加载，如果是一个成员方法，则会在外部实例化的时候加载。

### 匿名内部类

对于上述的方法内部类，也可以改写成如下的匿名内部类：

```java
public class Outer {
    interface People {
        void eat();
    }
    
    public People fun() {
        return new People() {
            @Override
            public void eat() {
                System.out.println("a people eat");
            }
        };
    }
}
```

可以看到，首先，匿名内部类没有名字，只有对抽象方法的实现，匿名内部类与方法内部类的功能是相似的，都是为了完成某个功能而创建的一个辅助类，但是这个类不需要被外界知道，外界只需要知道这个类是对某个抽象类/接口中抽象方法的一种实现，至于具体怎么实现的，外界不需要知道，也不需要知道到底是哪个类完成了这个实现。比如在 Android 中最常用的设置点击监听，我们就会常常用一种匿名内部类实现一个`OnClickListener`接口，然后对于外界而言，如果用户点击了这个按钮，它只需要调用这个监听器的对应方法即可，其它的都不需要知道。

在某些情况下，匿名内部类是内部类的一种缩写方式，当这个内部类只会用到一次，且对于外界而言，不需要知道这个类的具体名字的时候（外界只会用到这个类的父类名或者它实现的接口名），就可以直接使用匿名内部类，从而避免太多有名字的类影响编码的过程。

关于内部类，有几点需要注意：

1.  匿名内部类没有访问修饰符
2.  在匿名内部类中如果要使用外部的形参，那么这个参数必须是 final 的
3.  匿名内部类如果继承自一个抽象类，那么它需要沿用这个抽象类的构造方法

对于第一点很好理解，匿名内部类没有名字，所以在它存在的整个过程中只会被访问到一次，那就是定义它的时候，而访问修饰符是用来限制外界对这个类的访问的，但是对于一个连名字都没有的类来说，外界自然也就无从访问，访问限制也就不需要了。

为什么在匿名内部类中使用外部的参数时，一定要设置为 final ？这涉及到 Java 的闭包相关的概念，简而言之就是内部类在使用外部的参数的时候，使用的并不是原本的参数，而是原来参数的复制，只不过它们有相同的值而已。所以这就造成一个问题，如果我们单方面在外部/内部修改了某个参数的值的时候，这个值的更改只在外部/内部有效，而对应的内部/外部的值还是原来的值，这就造成了内外不统一的情况，所以，为了避免这种情况，Java 直接禁止了这种参数的修改，即给这种参数加一个 final 修饰符。但是在 Java 8 中好像不需要给参数增加一个 final 修饰符了，如下并不会报错：

```
public People fun(String s) {
    People p = new People() {
        @Override
        public String eat() {
            return s;
        }
    };
    return p;
}
```

但是在上述代码中增加了一句之后：

```
public People fun(String s) {
    People p = new People() {
        @Override
        public String eat() {
            return s;
        }
    };
    s = "fadf";
    return p;
}
```

还是会报错，报错内容为：`variable 's' is accessed from within inner class, needs to be final or effectively final`，从内部引用的本地变量必须是最终的或者是实际上最终的。也就是说，Java 8 并没有改变这个参数必须是 final 的情况，而是增加了一种判断：这个变量也可以是实际上 final 的，也就是这个变量不能被修改。

第三点，关于匿名内部类的构造方法，因为匿名内部类没有自己的名字，也就没有自己的构造方法，但是如果它继承的抽象类有构造方法，则它必须要延续父类的构造方法。如下：

```java
public class Outer {
    abstract class Man {
        String name;
        Man(String s) {
            name = s;
        }

        abstract void eat();
    }

    public Man fun2() {
        return new Man("") {
            @Override
            void eat() {
                System.out.println(name);
            }
        };
    }
}
```

### 静态内部类

静态内部类是与成员内部类相对应的一种内部类，就像是静态方法与成员方法之间的关系。上面对于成员内部类的讲述说道，成员内部类不能使用静态变量/方法是因为生命周期可能冲突的缘故，那么此时的静态内部类便没有那种限制了。静态内部类会在外部类加载的时候就加载（同静态变量/方法一致），所以静态内部类的实例化无需依赖于外部类的实例，所以静态内部类也就无法持有外部类的引用，也就导致它无法使用外部类的非变量/方法，这都是一连串的影响。

正是因为静态内部类不会持有外部类的引用，所以我们可以放心地使用这种内部类而不需要考虑因使用内部类而导致的内存泄漏等问题。在我看来，静态内部类对于外围类，是真正的完全独立的，就像是在同一个包下的两个类相似，当我们需要使用静态内部类的时候，一般也可以选择不定义在外部类内部而直接定义在包下。唯一的区别就是，我们可以给这个静态内部类加上访问限制修饰符，来决定这个类是否可以被其他类访问。（如果我们希望内部类只能被某一个类访问，那么我们应该将这个类定义为内部类，并加上 private 修饰符）

静态内部类一般如下定义：

```java
public class Outer {
    private String str;
    private void fun() {
        Inner.fun4();
    }
    
    public static void fun2() {
        
    }
    
    static class Inner {
        public void fun3() {
            fun2();
        }
        
        public static void fun4() {
            fun2();
        }
    }
}
```

然后在外界使用静态内部类的时候，除了需要使用外部类索引到内部类之外，也与外部类没有什么关系了：

```java
public class Main {
    public static void main(String[] args) {
        Outer.Inner.fun4();
        Outer.Inner inner = new Outer.Inner();
    }
}
```

### 内部类的使用

对于上面四种内部类都有了一定的介绍，那么为什么要使用内部类？应该在什么时候使用内部类？使用哪一种内部类呢？

首先，对于非静态的内部类而言，它可以直接访问外部类的所有成员变量/方法（有点像继承但又比继承的权限更大，且内部类与外部类是两个不同的对象），那么，如果这个内部类同时继承了另一个内部类的话，就意味着他可以同时拥有两个其他类的功能，这就有了一点「双重继承」的意思，如果再在这个内部类的里面增加一个内部类，继承于其他的类呢，就完成了「三重继承」，所以，内部类的一种功能就是完成类似于 Java 中不存在的「多重继承」的功能。这比使用接口在某些时候更加有效，因为要实现每个接口的每个抽象方法。

或者，如果要对一个类实现更深层次的隐藏，如只能被某一个类访问而不能被其他同一个包下的类访问，那就只能使用内部类这一种方法了，这是外部类所不具有的特性。

又或者，我们知道，接口或者抽象类只有被实现或继承的时候才能使用，那么有时候为了少写一些多余的代码，可能就需要匿名内部类了，它能够在不定义确定的类名的情况下直接通过实现抽象方法使用。

### 总结

内部类是 Java 提供的一种特殊的定义类的方式，它的存在能够完成一些 Java 本身不能提供的功能，如对类的更深的隐藏、某种程度上的「多重继承」或者是简化某些开发过程等。在开发中合理的使用内部类，有时能让我们的代码具有更强大的生命力。