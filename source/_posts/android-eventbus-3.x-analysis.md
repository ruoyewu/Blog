---
title: Android EventBus 3.x 分析
date: 2019-04-08 15:45
tags:
	- android
---

EventBus 是 Android 中一个用于在组件之间传递时间的解决方案，基于观察者模式的思想，与 Android 中的本地广播有类似的实现原理，不过相比之下更加自由，在这种方案下，通信的组件之间没有耦合。在一般的组件通信中，大致有两种方式，第一种是利用回调方法，在这种方法中，事件处理者始终需要持有一个回调接口，一方面不利于编写，大量的回调必然会导致代码的凌乱，另一方面二者之间通过接口产生耦合。第二种是使用广播通信，但是在 Android 中广播的使用需要实现 BroadcastReceiver 类，然后利用广播的一套通信机制完成组件之间的通信，这种情况下确实也可以消除组件之间的耦合，但是使用广播的有一定的限制（只能使用 Intent 传递数据），还有一点就是广播的使用还是比较麻烦的，相比之下回调接口还显得更加清爽。

而 EventBus 提供的组件之间通信的方案，可以很轻松的在原有基础上完成更改，只需要在订阅者中增加一个方法（方法的参数为一个事件），再为其添上一个注解，一个订阅者就完成了，再将其注册到事件总线上，那么在其他任何地方发布一个事件之后，事件就会顺着某一条线传递到每一个订阅者上，并调用这个方法。

整个框架的结构如下：

![](https://blog-1251826226.cos.ap-chengdu.myqcloud.com/EventBus-Publish-Subscribe.png)



在这个结构中包括四个部分：

1.  事件发布者
2.  事件订阅者
3.  事件总线
4.  事件

### EventBus 使用

首先，在 EventBus 中事件就是一个类，一个类似于 Message 类这样的传递数据的类，比如下面：

```java
public class EventMessage {
    public String msg;
    
    public EventMessage(String msg) {
        this.msg = msg;
    }
}
```

这是一个很简单的事件类，里面保存了一个字符串。

事件订阅者，一般是 Activity 或 Fragment ，以一个 Activity 为例：

```java
public EventBusActivity extends AppcompatActivity {
    @Override
    protected void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
    
    @Subscribe(threadMode = ThreadMode.MAIN, priority = 0, sticky = false)
    public void onEventMessage(Message msg) {
        // do something with Message
    }
    
    @Override
    protected void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
}
```

EventBus 就是事件总线类，据其使用方式可以看出这是一个单例，有一点需要注意，订阅者使用的时候一定是要注册和注销对应的，如果只有注册而没有注销的话，就会导致内存泄漏（在 EventBus 中会始终持有这个 Activity 的引用导致其无法被回收）。订阅者需要订阅的事件，这个事件就是由`onEventMessage()`方法确定的，所有的订阅方法都需要使用 Subscribe 标注，同样的，所有被 Subscribe 标注的一定是订阅方法，这个注解有三个字段，分别是 threadMode（执行这个方法的线程）、priority（优先度）、stricky（是否为粘性事件），方法有且只能有一个参数，这个参数就是事件。

然后是发布事件，发布事件只需要调用`EventBus.getDefault().post(Object)`即可，可以在任何需要发布事件的地方调用，如网络请求完成之后、IO 操作完成之后等子线程中，也可以是另一个 Activity 中，理论上，只要是同一个进程，任何地方都可以。事件发布者发布了一个事件（这里的 Object 对象），然后这个事件在 EventBus 中根据事件类型找到对应的订阅者，并调用订阅者中的对应的方法，例如在上述例子中，如果调用了`EventBus.getDefault().post(new EventMessage("msg"))`方法发布事件，最终会调用 EventBusActivity 的`onEventMessage()`方法，二者以事件 EventMessage 为联系，发布者发布的 EventMessage 事件被 EventBusActivity 接收到，并调用回调方法处理这个事件。

以上就是 EventBus 的基本使用，看起来很方便，仅需要极少的步骤，就可以完成各个地方事件的传递，那么这样一个功能是如何实现的？为什么给一个类加上一个带有注解的方法之后它就成为了一个订阅者？下面将根据 EventBus 3.1.1 的源码探索它的实现原理。

### 订阅

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

要注册一个订阅者很简单，只需要调用`register()`。上面是这个方法的实现，总得分为三步：

1.  得到订阅者的 Class 类型
2.  根据 Class 得到订阅者中的订阅方法
3.  将所有的订阅方法注册到事件总线中

取得订阅者的 Class 类型，是为了获得订阅方法作准备，通过调用`findSubscriberMethods()`获得了这个类的 SubScriberMethod 列表。再看这个方法的实现：

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

获取订阅方法也分为三步：

1.  从缓存池中寻找缓存
2.  使用反射获取订阅方法
3.  使用 SubscriberInfo 获取订阅方法

当同一个类的多个对象重复订阅的时候，它们的订阅方法信息就会在第一次获取之后被 METHOD_CACHE 缓存，它是一个 ConcurrentHashMap ，能够处理多线程的操作，由于缓存池的存在，每个类的反射只会执行一遍。如果没有缓存，就会直接查找，查找有两种方式：反射和 SubscriberInfo 。

先看反射，

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findUsingReflectionInSingleClass(findState);
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```

FindState 是 SubscriberMethodFinder 中的工具类，每一个 FindState 负责一个类（包括其父类）中订阅方法的查找，且 FindState 是从缓存池中获取的，类似于 Message 的获取，只不过这个缓存池只有默认 4 的大小。接下来是一个 while 循环，在循环中，首先调用`findUsingReflectionInSingleClass()`查找一个类的订阅方法，然后再调用`moveToSuperclass()`方法移动到这个类的父类再次查找。

利用反射查找订阅方法的逻辑如下：

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

过程也挺简单的，首先利用反射得到类中声明的方法，然后依此判断每一个方法，当满足一下几个条件就可以初步判断这是一个订阅方法：

1.  方法限定符为 public 且非 abstract 、static 、synthetic、bridge
2.  方法只有一个参数
3.  拥有 Subscribe 注解

当满足这些条件，就会调用`checkAdd(Method, Class<?>)`判断这个方法对于当前的类来说是否合法。

```java
boolean checkAdd(Method method, Class<?> eventType) {
    // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
    // Usually a subscriber doesn't have methods listening to the same event type.
    Object existing = anyMethodByEventType.put(eventType, method);
    if (existing == null) {
        return true;
    } else {
        if (existing instanceof Method) {
            if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                // Paranoia check
                throw new IllegalStateException();
            }
            // Put any non-Method object to "consume" the existing Method
            anyMethodByEventType.put(eventType, this);
        }
        return checkAddWithMethodSignature(method, eventType);
    }
}
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
    methodKeyBuilder.setLength(0);
    methodKeyBuilder.append(method.getName());
    methodKeyBuilder.append('>').append(eventType.getName());
    String methodKey = methodKeyBuilder.toString();
    Class<?> methodClass = method.getDeclaringClass();
    Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
        // Only add if not already found in a sub class
        return true;
    } else {
        // Revert the put, old class is further down the class hierarchy
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }
}
```

首先，根据 eventType （事件类型）判断这个类中是否已经有过当前事件的订阅方法，如果没有，则直接加入。如果已经有过这种事件的订阅方法，就通过`checkAddWithMethodSignature()`方法进行更精确的判断，在这个方法里会根据方法名和事件名生成一个 key ，所以如果两个订阅方法订阅了同一个事件但是两个方法名并不相同，这也是被允许的，但是一般情况下应该不会出现同一个类中两个方法订阅同一个事件的情况，这好像没有太多的意义。其次，在这个方法中还进行了父类的判断，如果一个类的父类中含有同样的订阅方法，那么从继承的角度来说，父类的方法应该会被子类覆盖，所以这里也做了相应的处理，如果两个类拥有同一个订阅方法，那么只会保留子类的订阅方法。

不过个人感觉在`checkAdd()`方法中有一点小 bug ，主要就是在下面这段：

```java
if (existing instanceof Method) {
    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
        // Paranoia check
        throw new IllegalStateException();
    }
    // Put any non-Method object to "consume" the existing Method
    anyMethodByEventType.put(eventType, this);
}
```

当某个事件在这个类（以及其父类）中的订阅方法中第一次遇到的时候，会将其加入 anyMethodByEventType ，existing 为 null ，当第二次遇到时，existing 为上一个方法，此时会先调用`checkAddWithMethodSignature()`方法，在这个方法里会将事件及方法所在类加入 subscriberClassByMethodKey ，然后就会将 anyMethdByEventType 中这个事件对应的 value 置为 this ，就是非 Method 对象。当第三次遇到的时候，由于 existing 是 this 而不是 Method 对象，就会跳过这段代码。第四次遇到的时候，existing 就变成了第三次的 Method 对象，此时仍就会进入这个代码段，如果`checkAddWithMethodSignature()`返回 false ，就会抛出 IllegalStateException 异常。

根据`checkAddWithMethodSignature()`的功能可以知道，如果一个类 A ，有三个父类，依此为 B C D，继承关系为 D->C->B->A ，在这四个类中定义了同样的一个订阅方法，那么在解析类 A 的订阅方法的时候，existing 不为 null 的情况有三次，第一次为 A 的方法，第二次为 this ，第三次为 C 的方法，在第三次 existing 不为 null ，也就是解析类 D 的时候，会抛出异常。

按我对这段代码的理解，我想作者的原意应该是想在第一次 existing 不为 null 的时候，才会进入上面这段代码，然后将第一次已经注册过的方法加入到 subscriberClassByMethod 中，然后在 anyMehodByEventType 中将这个事件对应的值置为 this ，然后在之后的解析中，都会不在此进入上面这个代码段，而是直接调用`checkAddWithMehodSignature(method, eventType)`判断这个方法是否可以添加。

但是在这里会出现一个问题，在`checkAdd()`方法中首先执行的是`Object existing = anyMethodByEventType.put(eventType, method)`，这个方法会覆盖`anyMethodByEventType.put(eventType, this)`，所以下次再调用`checkAdd()`方法的时候 existing 仍然是一个 Method 对象，这就会导致重复进入这个方法，并导致抛出异常。

只不过这种情况只会在连续四个祖孙类拥有同样的订阅方法的时候才会出现，而且我也可能并不理解作者在这里真正的意图，可能这样一个限制正是作者需要的。

话说话来，当判断完成之后，如果这个方法是合法的，就将其加入 FindState 的 subscriberMethods （订阅方法链表）中，最终完成订阅方法的查找。

上面是反射查找的过程。EventBus 还提供了另一种查找方式，通过 SubscriberInfo 查找。

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

这种方法与反射查找的方式相比显然会快不少，但是这种方式的使用比较复杂，当使用反射的时候只需要给一个方法加上 Subscribe 注解就能使其成为订阅方法，所有的中间生成数据由 EventBus 利用反射完成，而这种方式需要开发者自己完成中间数据的生成，说白了就是由开发者手动写出来，一个标准的过程如下：

```java
EventBus.builder()
        .addIndex(new SubscriberInfoIndex() {
            @Override
            public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
                if (subscriberClass == EventBusActivity.class) {
                    return new SubscriberInfo() {
                        @Override
                        public Class<?> getSubscriberClass() {
                            return subscriberClass;
                        }
                        @Override
                        public SubscriberMethod[] getSubscriberMethods() {
                            try {
                                Method m1 = subscriberClass.getMethod("onEventMessage",
                                        EventBusActivityP.Message.class);
                                return new SubscriberMethod[] {
                                        new SubscriberMethod(m1, EventBusActivityP.Message.class, ThreadMode.MAIN, 0, false)
                                };
                            } catch (NoSuchMethodException e) {
                                e.printStackTrace();
                            }
                            return null;
                        }
                        @Override
                        public SubscriberInfo getSuperSubscriberInfo() {
                            return null;
                        }
                        @Override
                        public boolean shouldCheckSuperclass() {
                            return false;
                        }
                    };
                }
                return null;
            }
        })
        .build();
```

实现 SubscriberInfoIndex 接口，为每一个类都编写好它的 SubscriberMthod 对象，EventBus 在使用的时候直接调用，使用这种方式的话就不需要 Subscribe 注解了，但是使用起来非常麻烦也是真的。还有，使用这个也就不能直接使用`EventBus.getDefault()`获得实例了，默认的实例是没有使用 SubscriberInfoIndex 的。

利用上面两种方式得到订阅方法之后，下一步就是注册到事件总线中，使用的是`subscribe()`方法。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

步骤如上，总得分为三步。第一步，根据事件类型找到所有的订阅者，并将当前的订阅者按照优先级插入到订阅者列表中。第二步，根据订阅者找到它订阅的所有事件，并将事件插入到这个订阅者的时间列表中，这一步与上一步完成的工作就是，订阅者、订阅事件这二者之间可以互相查找。第三步是关于粘性事件的操作，如果当前订阅者接收粘性事件，就将粘性事件发布给它。

订阅者的订阅过程到这里就结束了，过程的话，总得也就是两步，首先根据订阅者找到它的所有订阅方法，然后将所有的订阅方法注册到 EventBus 中。

### 发布

订阅完成之后就可以发布事件了，发布事件的入口为`post()`方法，当然还有发布粘性事件的`postSticky()`方法，下以`post()`为例查看整个发布过程：

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

事件的发布也用到了一个工具类，PostingThreadState ，这是一个线程私有工具，每一个线程中只有一个在工作，当有新的事件要发布时，首先将其加入事件队列，然后利用 PostingThreadState 对整个队列的事件依此发布，直到事件队列为空才停止发布。所以当当前线程的 PostingThredState 正在发布的时候，只需要将其入队即可，接着调用`postSingleEvent()`。

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

由于事件也是以类的形式存在的，那么当一个事件是另一个事件的子类的时候，发布这个子类对应的事件，父类事件的订阅者是否能收到这个事件？还有，如果一个事件没有订阅者怎么办？本方法解决的就是这样一个问题，如果标识 eventInheritance 为 true ，就将这个事件也发送给这个事件父类的订阅者。如果没有订阅者接收这个事件，就再做处理。

具体的发送步骤在`postSingleEventForEventType()`中。

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

找到这个事件的所有的订阅者，并调用`postToSubscription()`发送到具体的订阅者。

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

这里就可以看到 ThreadMode 起到的作用，总共有五类 ThreadMode ：

1.  POSTING，直接在发布线程调用
2.  MAIN，在主线程调用，如果发布线程就是主线程，则会阻塞主线程直接调用
3.  MAIN_RODERED，在主线程非阻塞地调用，内部采用 Handler 将其发送到 MessageQueue 中等待执行
4.  BACKGROUND，在后台线程调用：如果当前线程非主线程，直接调用，如果是主线程，则将其加入到正在执行的后台线程中执行，也就是说，一个事件发布之后，需要等其他事件订阅者处理完 BACKGROUND 的事件之后才能轮到当前事件的发布
5.  AYSNC，每次都直接交给一个子线程调用，直接交给线程池一个任务，等待线程池的调用

处理的逻辑就是这么个逻辑，最终还是会调用`invokeSubscriber()`方法，调用订阅者的订阅方法。

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

处理很简单，直接利用反射调用方法即可。

至此，一个事件，就从发布者传递到了订阅者，订阅者收到事件，调用对应的订阅方法处理这个事件。

### 注销

当一个订阅者不需要了之后，需要及时调用注销方法将订阅者注销掉，否则就会导致内存泄漏。注销方法是`unregister()`。

```java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

首先根据订阅者找到这个订阅者订阅的所有事件，然后调用`unsubscribeByEventType()`，对每一个事件进行注销。

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

然后再根据事件找到每一个事件的所有订阅者，在这些订阅者里面找到需要移除的订阅者，将这个订阅解除。这就注销完成了。

### 总结

纵观 EventBus 的源码，除了其提供的方便快捷的组件通信的功能外，其源码的编写上也有一些可以借鉴的地方。

关于 EventBus 的创建，使用的是建造者模式，对应的类就是 EventBusBuilder ，使用建造者模式带来的优点就是对 EventBus 的封装。EventBus 中的参数基本上都只是在构造的时候传递的，而创建完成之后这些参数不可修改，这能够防止在 EventBus 运行过程中修改参数带来的不可控因素，但是 EventBus 在创建的时候需要很多参数，如果将这些参数都放入构造方法中无疑是比较复杂的，所以就有了 EventBusBuilder 类，它是 EventBus 的建造者，在它的内部设置了默认的参数，而且开发者可以调用它的方法设置参数，最终调用`build()`建造出一个 EventBus 实例，开发者只能在建造 EventBus 之前设置参数，建造完成之后就不能修改，并且构造的时候利用 EventBusBuilder ，开发者只需要修改需要修改的参数即可。

关于 EventBus 的使用，可以通过`EventBus.getDefault()`得到一个实例，说明这是一个单例模式的应用，在 EventBus 中为开发者内置了一个默认的实例，使用的是默认的参数，如果没有特别的需求，默认的这个实例也就够用了，一方面避免了过多的实例存在，一方面也简化了 EventBus 的使用。

再说 EventBus 本身，它是一个基于观察者模式思想的框架。观察者模式最重要的功能就是解耦，将责任双方分离成观察者和被观察者，虽然观察者能观察到被观察者，但是二者之间没有直接联系，它们通过将自身注册到总线上，来完成信息的传送。EventBus 以事件为驱动，将相关的类划分为订阅者和发布者，二者通过事件联系起来，订阅者订阅某种事件，当发布者发布一个事件的时候，这个事件会被发布给每一个这个事件的订阅者。在 Android 系统这样一个拥有界面的系统中，组件之间的通信非常频繁，一般的处理逻辑就是，用户点击屏幕->应用接收到操作请求->执行请求->返回请求结果并展示。所以几乎每一个操作都需要对应的工作者，而工作者处理的结果，一般是使用回调接口返回到 UI 上。而如果使用基于观察者模式的 EventBus ，那么 UI 接收到用户请求之后便可以直接发布一个请求事件，具体由谁执行就要看这个事件的订阅者是谁了，工作完成之后就可以再发布一个结果事件，这个事件由 UI 订阅，收到之后再进行展示。这样的好处就是，请求的发出者与请求的执行者，二者之间没有联系，互相并不知道谁是谁，它们只需要专注于自己的任务，收到事件就处理，处理完了再发布结果，完全不需要知道其他组件，自然组件之间也就没有耦合。

但是使用 EventBus 也是有一定的缺点，由于它基于事件工作，那么当系统足够复杂的时候，事件类型就必然也特别多，就会导致类数量的增多，当事件类型很多的时候，由于各组件之间没有联系，仅靠事件类型关联，那么对于开发者查看源码理解这个系统来说就不那么友好了。过多的事件类型也会给编码带来一定困难，有时开发者可能找不到到底该用什么事件。