---
title: 设计模式六大原则
tags: 
	- design pattern
---

## 设计模式

## 六大原则

### 单一职责原则

**Single Responsibility Principle**

#### 定义

不要存在多于一个导致类变更的原因，即一个类只负责一项职责。

#### 描述

在最初新建一个类的时候，一般都能够保持「单一职责原则」，但是一个类建立完成之后一般不是一成不变的，往往随着应用规模的提升，一个类的功能可能会慢慢地增加，增加到一定程度之后，可能就不符合「单一职责原则」了，比如有一个例子：

最初，我们需要构建一个类，来表示一个动物需要吃的东西，如一个肉食动物需要吃肉：

```java
class Animal {
  private String animalName;

  public Animal(String animalName) {
    this.animalName = animalName;
  }
  
  public String food() {
    return animalName + " eat meat";
  }
}
```

我们完全可以这样定义，当我们构建了一个新的 Animal 对象的时候，如果是一个鳄鱼，那么调用`food()`方法就会返回`鳄鱼 eat meat`，但是并不是所有的动物都是肉食动物，很显然这个`food()`方法不能满足全部的需求，那么就要对这个类进行整改，整改的方法也是多种多样的，比如以下几种：

1.  在原本的方法内部增加一部分判断

    ```java
    class Animal {
      private String animalName;
      
      public Animal(String animalName) {
        this.animalName = animalName;
      }
      
      public String food() {
        // 需要另外定义两个方法来判断某个动物是肉食动物还是素食动物
        // isCarnivorous() 是否为肉食动物
        // isVegetarian() 是否为素食动物
        if (isCarnivorous(animalName)) {
          return animalName + "eat meat";
        }else if (isVegetarian(animalName)) {
          return animalName + "eat vegetables";
        }
      }
    }
    ```

2.  将 Animal 抽象出来，然后定义不同的类完善这个类。

    ```java
    abstract class Animal {
      protected String animalName;
      
      public Animal(String animalName) {
        this.animalName = animalName;
      }
      
      public abstract String food();
    }

    class CarnivorousAnimal extends Animal {
      public String food() {
        return animalName + ' eat meat';
      }
    }

    class VegetarianAnimal extends Animal {
      public String food() {
        return animalName + " ear vegetables";
      }
    }
    ```

    如上，以上两种方法，都解决了上述的问题，使得对于食肉动物与素食动物，都能返回其正确的食物类型，但是对于第一种方法来说，我们只是在原先的类的基础上进行了一定的修改，并修改的方法内部的代码，很显然这并不符合单一职责原则，而且在将来如果需要添加其他种类的动物的时候，比如说杂食动物，还需要再次对`food()`方法进行修改，这样也很容易导致未来的某一个时刻，出现了肉食动物吃素的情况。对于第二种方法来说，我们将 Animal 这个类抽象化，只保留了它基本的功能，但对于`food()`这个方法来说，它并没有给出具体的实现，而我们要使用这个类的时候，需要再构建一个类，比如 VegetarianAnimal ，素食动物类，实现的`food()`方法就可以不需要多余的判断直接返回`eat vegetables`了，而且当应用规模扩展，需要使用到杂食动物的时候，只需要新建一个 OmnivoreAnimal 继承 Animal 类并实现根据自己的情况实现`food()`这个方法即可，不需要对原有的代码作任何的修改，以后再增加更多也是这样的一个逻辑，开发者只需要对当前新建的内容负责，而不需要考虑新建的内容对原有内容是造成影响带来的风险。

#### 作用

1.  降低类的复杂度
2.  提高类的可读性
3.  变更导致的风险降低

### 里氏替换原则

**Liskov Substitution Principle**

#### 定义

对于程序中任何一个使用 P 的对象的地方，都可以使用 C 的对象来替换，则 C 是 P 的子类，即所有引用基类的地方必须能透明地引用其子类的对象，即**子类可以扩展父类的功能，但不能改变父类原有的功能**。

#### 描述

**继承**是面对对象语言的一个大优势，能够使类与类之间的关系丰富多样，可以随时在不改变原有的一个类的基础上为其扩展不同的功能。但是这样也有一些危险，比如当子类重写了父类的某项功能之后，在使用中子类仍然可以无缝替换父类，但是子类对父类的某项功能的改变必定会增加程序出错的风险，所以就有了「里氏替换原则」。「里氏替换原则」是麻省理工学院的一位女士「Barbara Liskov」提出的，专门用来限制这个问题。举个例子，就拿上面的例子来说吧，比如业务规模又增加了，单纯的食肉动物也不能表达客户需要，还要在这个基础上增加一个水、陆、空的食肉动物，这样的话就可以新建一个类：

```java
class LandCarnivorousAnimal extends CarnivorousAnimal {
  public String food() {
    return "meat";
  }
  
  public String field() {
    return "land";
  }
}
```

可以看到，LandCarnivorousAnimal 对其父类 CarnivorousAnimal 的`food()`	方法进行了重写，将原本应该返回`animalName + "eat meat"`的返回结果变为了`"meat"`，但是当使用的时候，很有可能会忽略了这个方法被重写的事实，使用者使用这个子类依旧按照使用父类的方法使用，就会不可避免地导致程序出错了。

#### 作用

1.  降低程序出错的概率
2.  保证父类的功能不会被子类覆盖

### 依赖倒置原则

**Denpendence Inversion Principle**

#### 定义

高层模块不依赖低层模块，二者都依赖其抽象。抽象不依赖细节，细节依赖抽象。

#### 描述

依赖倒置原则的基本思想就是，降低类之间的耦合程度，将一个类 A 依赖的实体类 B 、C 等，尽可能地转变为依赖一个接口 I ，然后让 B 、 C 实现这个接口 I ，这样类 A 就与 B 、C 没有依赖关系了，这样如果要对这个逻辑进行修改，就只需要新建一个类实现接口 I ，看实例：

```java
class Reader {
  public String readBook(Book book) {
    return "read " + book.getContent();
  }
  
  public String readPaper(Paper paper) {
    return "read " + paper.getContent();
  }
}

class Book {
  public String getContent() {
    return "this is a book";
  }
}

class Paper {
  public String getContent() {
    return "this is a paper";
  }
}
```

如上，这是一个实现了阅读者阅读读物的逻辑，可以看到 Reader 类直接依赖了 Book 类和 Paper 类，在这里 Reader 就是那个高层的负责业务逻辑的类，很显然 Reader 依赖的类过多了，并且如果需要增加新的读物，还需要修改 Reader 类，使其依赖更多的类才能完成这个逻辑。如果使用依赖倒置原则，使 Reader 依赖于一个抽象 IReading ，然后使 Book 和 Paper 实现 IReading ，就可以 Reader 中依赖过多的问题：

```java
class Reader {
  public String read(IReading reading) {
    return "read " + reading.getContent();
  }
}

interface IReading {
  String getCotnent():
}

class Book implements IReading {
  public String getContent() {
    return "this is a book";
  }
}

class Paper implements IReading {
  public String getContent() {
    return "this is a paper";
  }
}
```

使用这种方法就可以减少 Reader 的依赖，使得 Reader 与 Book 和 Paper 没有直接依赖关系，它们通过接口 IReading 间接联系到了一起。

#### 作用

1.  降低程序的耦合度
2.  提高程序的编译效率

### 接口隔离原则

**Interface Segregation Principle**

#### 定义

一个类对另一个类的依赖应该建立在最小的接口上，即一个客户端不应该依赖它不需要的接口。

#### 描述

接口隔离，即对于一个实现某个接口 I 的类 A 来说，I 中定义的需要实现的方法应该都是 A 中能够用到的，不然就会出现 A 实现了 I 中一个它不会用到的方法，这样显然会造成 A 的负担，也会增加系统的安全性问题。 所以「接口隔离原则」的目标就是，将一个功能比较复杂的接口细化成完成某个单一职责的接口，但是这会有一个问题，就是在细化的时候，要把握好这个细化的度，不能使得接口过多，这样同样会增加系统的负担，还会增加开发人员的负担。

同时，「接口隔离原则」与「单一职责原则」的区别在于，「单一职责原则」注重的是对一个类的职责的划分，针对的是程序中实现和细节，而「接口隔离原则」注重的是对接口依赖的隔离，针对的是抽象和程序的整体框架。

#### 作用

1.  提高类的内聚
2.  提高程序安全性

### 迪米特法则

**Law of Demeter**

#### 定义

一个对象应对其他对象保持最少的了解，即耦合度尽可能低。

#### 描述

总所周知，类与类之间的联系越紧密，程序的耦合性就越高，类与类之间的影响也就越大，一个类的改变，导致其他类出错的概率也就越高。所以「迪米特法则」的目标就是，尽量降低类与类之间的耦合。

「迪米特法则」的另一个定义为：「只与直接的朋友通信」。所谓「直接的朋友」，就是指在当前类中以成员变量、方法参数、方法返回值出现的类。「只与直接朋友通信」就是指，除了上面的这些「直接的朋友」以外，当前类中不应该出现以「局部变量」的形式出现的其他的类，如果我们确实需要相应的功能，应该通过「直接的朋友」调用它的「直接的朋友」的方式，来使用相关的功能。一个简单的例子，有一个公司，它有自己的职员，还有自己领导的分公司，同时分公司也有自己的职员。

```java
class Employee {
  private String id;
  
  public void setId(String id) {
    this.id = id;
  }
  
  public String getId() {
    return this.id;
  }
}

class SubEmployee {
  private String id;
  public void setId(String id) {
    this.id = id;
  }
  
  public String getId() {
    return this.id;
  }
}

class SubCompany {
  private List<SubEmployee> employeeList;
  
  public void getEmployee() {
    return this.employeeList;
  }
}

class Company {
  private List<Employee> employeeList;
  
  public void printEmployee(SubCompany company) {
    for (SubEmployee e : company.getEmployee()) {
      System.out.println(e.getId());
    }
    
    for (Employee e : employeeList) {
      System.out.println(e.getId());
    }
  }
}
```

对于上面的类，可以看到，如果总公司需要打印包括分公司在内的所有职员的 Id ，就需要依赖类 SubEmployee ，但是类 SubEmployee 并不是类 Company 「直接的朋友」，所以这就违反了「迪米特法则」，可以对上面的代码进行一定的修改：

```java
class SubCompany {
  private List<SubEmployee> employeeList;
  
  public void getEmployee() {
    return this.employeeList;
  }
  
  public void printEmployee() {
    for (SubEmployee e : employeeList) {
      System.out.println(e.getId());
    }
  }
}

class Company {
  private List<Employee> employeeList;
  
  public void printEmployee(SubCompany company) {
    company.printEmployee();
    
    for (Employee e : employeeList) {
      System.out.println(e.getId());
    }
  }
}
```

将 SubCompany 与 Company 进行一定的修改，就可以使之符合「迪米特法则」，但是可以看到，如果 SubCompany 本身不需要使用`printEmployee()`方法，而是仅仅为了 Company 新建了一个方法显然有些浪费，同时如果大量出现类似的情况，肯定会大幅增加程序中多余代码的量。所以使用「迪米特法则」的时候，应该再三考量，得到一个最有效的解决方案。

#### 作用

1.  降低类之间的耦合度

### 开闭原则

**Open Close Principle**

#### 定义

一个软件实体，如类、模块和方法，应该对扩展开放，对修改封闭。

#### 描述

在一个软件的生命周期中，需求总会在不停地改变，所以如果在修改功能的过程中，不加限制地修改原有的代码，肯定会增加原有的代码也变得不可用的概率，到时候只能对所有代码进行重构，所以「开放封闭原则」规定，只允许扩展原有代码的功能，而不对其进行修改。

## 参考

[http://www.uml.org.cn/sjms/201211023.asp](http://www.uml.org.cn/sjms/201211023.asp)

[http://wiki.jikexueyuan.com/project/java-design-pattern-principle/](http://wiki.jikexueyuan.com/project/java-design-pattern-principle/)