---
title: Android Lifecycle 简介
date: 2019-04-30 14:00
tags:
	- android
---

### 基本功能

Lifecycler 是一个生命周期管理工具，说的具体点，是用于管理 Activity 和 Fragment 生命周期的工具，它可以探测到 Activity 和 Fragment 这些宿主生命周期的变化，并将这些变化通知给需要的类。其使用场景，就是在某些时候，一个类需要在`onCreate()`中进行初始化，在`onDestroy()`中回收的时候，如果没有 Lifecycle ，就需要在每一个使用到这个类的 Activyt/Fragment 的对应的生命周期的方法中调用类的方法，一来会增加 Activity/Fragment 的复杂度，而来也会复杂化这个类的使用，而使用了 Lifecycle 之后，一个类就可以监听到 Activity/Fragment 生命周期的变化自行作出对应操作，而只需要在 Activity/Fragment 中增加一句代码：`getLifecycle().addObserver()`。

Lifecycler 是一个抽象类，声明如下：

```java
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    
    @MainThread
    @NonNull
    public abstract State getCurrentState();
    
    public enum State {
        DESTROYED, INITIALIZED, CREATED, STARTED, RESUMED;
        
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
    
    public enum Event {
        ON_CREATE, ON_START, ON_RESUME, ON_PAUSE, ON_STOP, ON_DESTROY, ON_ANY
    }
}
```

### 生命周期获取

可以看到，Lifecycle 可以拥有许多的观察者，这些观察者可以观察到宿主的生命周期，并在合适的生命周期中作出相应的操作，它的实现类是 LifecycleRegistry 。那么第一个问题，Lifecycle 是如何得到宿主的生命周期的？

首先看其构造方法，

```java
public LifecycleRegistry(@NonNull LifecycleOwner provider) {
    mLifecycleOwner = new WeakReference<>(provider);
    mState = INITIALIZED;
}
```

在初始化一个 LifecycleRegistry 的时候需要一个 LifecyclerOwner ，根据类名就可以看出，这应该就是宿主，LifecycleOwner 是一个接口：

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

这个接口的实现类，应该就是生命周期的实际提供者，也就是 Activity 和 Fragment 。首先看 Fragment ，在 androidx.fragment.app 包下的 Fragment 实现了这个接口，并且在`initLifecycle()`方法中初始化了 LifecycleRegistry ：

```java
private void initLifecycle() {
    mLifecycleRegistry = new LifecycleRegistry(this);
    
    // ...
}
```

并且，在 Fragment 的各生命周期回调方法`performXXX()`中，都会调用 LifecycleRegistry 的`handleLifecycleEvent()`方法将生命周期传递过去。如`performCreate()`方法：

```java
void performCreate(Bundle savedInstanceState) {
    // ...
    
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
}
```

由此可以看到，Fragment 通过在自己的生命周期中调用`handleLifecycleEvent()`，将事件传递给了 Lifecycle ，这个方法就是 Lifecycle 获取生命周期的入口。

上面讲到的是 Fragment 中 Lifecycle 获取生命周期的过程，在 Activity 中这个过程是类似的，不过也有所不同。首先说明一点，在 Activity 中其生命周期的获取是通过 Fragment 实现的，每一个 Activity 都会被添加一个 Fragment ，由于 Fragment 的生命周期是与 Activity 保持一致的，所以可以直接使用这个 Fragment 作为宿主，这个 Fragment 就是 RecordFragment 。

基本思想是这样的，给所有的 Activity 都增加一个 RecordFragment ，在 RecordFragment 有对其生命周期的监听，没当回调一个生命周期的方法时，就将当前的生命周期散布出去（这是 RecordFragment 中实现的逻辑），因为 Fragment 与 Activity 的生命周期是一致的，所以使用这种方法可以获得 Activity 的生命周期。

下面的问题就是，Fragment 是在什么时候被添加到 Activity 中的？关于添加 RecordFragment ，具体的逻辑在`injectIfNeededIn()`中：

```java
public static void injectIfNeededIn(Activity activity) {
    // ProcessLifecycleOwner should always correctly work and some activities may not extend
    // FragmentActivity from support lib, so we use framework fragments for activities
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```

可以初步确定这个方法调用的时候就是给 Activity 添加 Fragment 的时候，有两个地方会调用这个方法，一个是在 ComponentActivity 中，另一个是在 LifecycleDispatcher 中。

除了在 ComponentActivity 中的`onCreate()`方法中会调用这个方法，还有一个相关的类，LifecycleDispatcher 中的 DispatcherActivityCallback 中的`onActivityCreated()`方法中也会调用这个方法，DispatcherActivityCallback 实现了 Application.ActivityLifecycleCallbacks 接口，这个接口是 Application 提供的用于获取相应的生命周期的接口，

```java
public interface ActivityLifecycleCallbacks {
    void onActivityCreated(Activity activity, Bundle savedInstanceState);
    void onActivityStarted(Activity activity);
    void onActivityResumed(Activity activity);
    void onActivityPaused(Activity activity);
    void onActivityStopped(Activity activity);
    void onActivitySaveInstanceState(Activity activity, Bundle outState);
    void onActivityDestroyed(Activity activity);
}
```

如上，所有的 Activity 的生命周期都在这个接口中有对应的回调，这是系统本身提供的一种 Activity 的生命周期的监听方法，具体的流程为，在 Activity 的回调方法`onCreate()`中有一句代码`getApplication().dispatchActivityCreated()`，然后在 Application 的这个方法里就会将这个生命周期事件传递给所有的 Callback ，而此时这样一个回调就可以被 Lifecycle 用来给所有的 Activity 注入 RecordFragment ，从而完成在 Lifecycle 这样一个机制中对 Activity 生命周期的监听。

但是光有这些基础还不够，最关键的是给 Application 添加上这个 Callback ，才能回调相关的方法，这可以很简单，在我们的应用中增加一个自定义的 Application ，然后在它的`onCreate()`方法中增加一句话即可。但是这样就意味着，开发者想要使用这样一个功能，首先要做的是实现一个 Application 再增加一句初始化代码，这显然不是一个好的解决方案。

所以 Lifecycle 的开发者没有使用上面的方法，而是通过 ContentProvider 完成这样一个初始化的过程，在 ProcessLifecyclerOwnerInitializer 这个类的`onCreate()`方法中，

```java
public boolean onCreate() {
    LifecycleDispatcher.init(getContext());
    ProcessLifecycleOwner.init(getContext());
    return true;
}
```

这是一个自定义 ContentProvider ，当应用打开之后会调用它的`onCreate()`方法，然后在这个方法中完成初始化，这个类随着 Lifecycle 共同存在，这就可以在不需要开发者手动初始化完成这样一个过程，在使用者看来，什么也不会看到，他只需要知道调用了 Activity 或 Fragment 的`getLifecycle()`方法就可以得到一个 Lifecycle 实例即可。

上面就是 Lifecycle 获取 Activity/Fragment 生命周期的过程，看起来很复杂。至于为什么要这么复杂？为什么在 Activity 中不能像 Fragment 中那样直接在`onCreate()`方法中调用 Lifecycle 的`handleLifecycleEvent()`方法？

首先要说明一点，Lifecycle 集成在 androidx 包中，而这个包只是为了兼容而提供的扩展包，而扩展包之所以存在，是因为老版本的系统即然已经发布了，那么这些已经发布的代码必然就是不可能再去更改的了，所以才需要这么复杂地实现这个功能。Activity 是系统的代码，不能修改，而 Fragment 已经说了，是 androidx 包下的一个 Fragment ，所以可以随心修改其代码。

那么既然如此，还有一点，ComponentActivity 也是 androidx 包下的，可以自由编写，又为什么不在这个类里写上与 Fragment 类似的功能？因为 Fragment 是重写的类，而 ComponentActivity 本质上还是继承了 Activity 的子类，所以只有开发者使用了 ComponentActivity 及其子类才能有这个功能，而如果开发者直接使用的是 Activity ，但是实现了 LifecycleOwner 接口，也实现不了这个功能，所以应该是为了兼容性，才有了 LifecycleDispatcher 这个类监听所有 Activity 的生命周期。

那么反过来，即然已经有了 LifecycleDispatcher 了，为什么还要在 ComponentActivity 的`onCreate()`方法中再初始化 RecordFragment ？首先 RecordFragment 的初始化方法保证了多次调用只生效一次，所以可以重复调用，至于为什么在 ComponentActivity 及其子类中会重复调用，可能是为了保证能够执行到吧。

### 内部事件传递

通过以上步骤，Lifecycle 能够获得到 Activity/Fragment 的各个生命周期，以`handleLifecycleEvent()`为入口，下面将会分析这个事件是如何传递给所有的观察者的。

需要 Lifecycle 提供生命周期事件的观察者首先要将自己注册到 Lifecycle 里，调用`addObserver()`方法。在这个方法里可以知道，LifecycleRegistry 内部有一个 FashInterableMap 用于存放所有的观察者，其 key 为 LifecycleObserver ，value 为 ObserverWithState ，当有一个新的观察者加入的时候，会按照当前的状态，依此将所有的事件发布给这个观察者，即如果当前的状态是 RESUMED ，然后此时加入了一个观察者，其初始的状态为 INITIALIZED ，那么此时会依此将 ON_CREATE、ON_START、ON_RESUME 这三个事件发布给观察者，使这个观察者立马到达当前的状态。对应的代码如下：

```java
while ((statefulObserver.mState.compareTo(targetState) < 0
        && mObserverMap.contains(observer))) {
    pushParentState(statefulObserver.mState);
    statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
    popParentState();
    // mState / subling may have been changed recalculate
    targetState = calculateTargetState(observer);
}
```

在 Lifecycle 中有 Event 和 State 这两种枚举量，其驱动过程如下：Lifecycle 从 Fragment 中接收到新的事件，然后改变自己的状态，接着向所有的观察者发布新的事件。当 Lifecycle 接收到一个事件之后，首先调用`getStateAfter()`方法得到当前应该处于的状态，接着调用`moveToState()`，将自身状态置为当前的状态，并将这个状态转化为事件发布给所有的观察者，在`sync()`方法中。

```java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

`sync()`的代码也不复杂，就是一个循环，这里涉及到 吗ObserverMap 的数据结构，它是一个类似于 LinkedHashMap 的结构，能够按照观察者的添加顺序成链，然后判断观察者的状态和当前实际状态之间是否有差，如果有差就需要更改观察者的状态，分别是`backwardPass()`（前向状态）和`forwardPass()`（后向状态），内部的逻辑也就是将所有的观察者的状态通过前递进或后递进的方式更改到当前状态，最终，它们都需要将观察者的状态转化为事件，发布出去，调用`dispatchEvent()`方法。

```java
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;
    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }
    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

在这个方法里，调用的是`mLifecycleObserver.onStateChanged()`，这是 LifecycleEventObserver 的方法，但是在添加观察者的时候添加的是 LifecycleObser ，可见在初始化的时候将原来的 LifecycleObserver 包装成了 LifecycleEventObserver ，包装调用的方法是`Lifecycling.lifecycleEventObserver()`，这个方法有些复杂，相关的涉及到 LifecycleObserver 的五个子类，分别是 FullLifecycleObserverAdapter、LifecycleEventObserver、SingleGenerateAdapterObserver、CompositeGeneratedAdaptersObserver 和 ReflectiveGenericLifecycleObserver ，其中前两个通过类型转换完成，后三个利用反射完成。

当添加的观察者是 LifecycleEventObserver 或 FullLifecycleObserver 的实现类的时候，直接通过类型转换就可以调用其`onStateChanged()`方法，否则就会利用反射，将原来的观察者进一步包装成 LifecycleEventObserver 的实现类，即后面的三个。

```java
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                (LifecycleEventObserver) object);
    }
    if (isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }
    if (isLifecycleEventObserver) {
        return (LifecycleEventObserver) object;
    }
    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}
```

可以看到，利用反射包装原 LifecycleObserver 有两种形式，一种是 GENERATED_CALLBACK ，另一种是 REFLECTIVE_CALLBACK ，就目前而言，开发者不会使用到 GENERATED_CALLBACK ，具体的表现为在`Lifecycling.generatedConstructor()`方法中总是会加载不存在的类导致不会返回 GENERATED_CALLBACK 这种类型。至于 REFLECTIVE_CALLBACK 这种方式，其基本使用如下：

```java
class Observer : LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun onAny(owner: LifecycleOwner, event: Lifecycle.Event) {
        
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreate() {
        
    }
}
```

开发者可以自行实现一个类继承 LifecycleObserver 接口，然后利用 OnLifecycleEvent 注解的方式，编写回调接口。如果添加的观察者是如上类的对象时，会在上面的包装过程中将其包装成 ReflectiveGenericLifecycleObserver 对象，在这个类中会对 Observer 类进行解析，找到其中使用了注解的方法，将其保存起来，并在周期回调的时候根据注解调用对应的方法。

对于其他的，观察者如果直接就是 LifecycleEventObserver 实现类，则直接返回，如果观察者是 FullLifecycleObserver 的实现类，则将其包装成 FullLifecycleObserverAdapter 类，因为 FullLifecycleObserver 并不是LifecycleEventObserver 子类，所以还需要进一步包装。

```java
interface FullLifecycleObserver extends LifecycleObserver {

    void onCreate(LifecycleOwner owner);

    void onStart(LifecycleOwner owner);

    void onResume(LifecycleOwner owner);

    void onPause(LifecycleOwner owner);

    void onStop(LifecycleOwner owner);

    void onDestroy(LifecycleOwner owner);
}
```

包装结果如下：

```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```

将这种方式与上述利用反射的方式进行对比之后便可以发现，这种是对反射的一种优化情况，可以直接调用方法而不是利用反射调用方法，但是这种方法并不能像使用注解那样灵活，要么只有一个`onStateChanged()`方法，要么有整个周期的全部方法，当然自己实现一个 LifecycleEventObserver 的子类只实现部分生命周期的方法也可以，不过那种情况这里不考虑。

总的来说，流程进行到这里就结束了，在什么周期需要做什么只需要去实现对应的方法即可。

### 总结

以上就是 Lifecycle 的整个实现原理，有两个比较复杂的地方，一个是 Activity 生命周期的获取，一个是生命周期回调的时候提供的多种方案的实现，不过就具体的事件传递的流程来说还是比较直观的，从 Activity/Fragment 处获取生命周期，然后再将这些生命周期告诉给所有的观察者。

那么为什么要这个大费周章一定要得到 Activity/Fragment 的生命周期呢，个人觉得，Activity/Fragment 是与用户交互的组件，其生命周期一方面代表的界面的显示情况，一方面代表着相关的资源的状态，在不同的生命周期执行不同的操作，才能更加合理地利用系统的资源，一个典型的例子就是内存泄漏，当一个功能需要持有 Activity 的引用的时候，一定需要在合适的时候（如`onDestroy()`中）释放引用，否则就会造成内存泄漏（当然还可以使用弱/软引用，这里不考虑），常规的方法是将其直接写到 Activity 中，但是这显然会造成 Activity 中代码量的增加、逻辑复杂度的增加，而使用 Lifecycle 的好处就是，这些功能类能够主动监听 Activity/Fragment 生命周期的变化而做出相应的操作，而不是等待着在 Activity/Fragment 中被动调用方法执行对应操作。

虽然在代码上来看这其中的区别不大（因为 Lifecycle 也是需要在 Activity/Fragment 中对应的生命周期中被调用的），但是从思想层面上看，Lifecycle 实现于系统内部，抛开其实现原理不谈，仅从使用者的角度看，那些原本需要在 Activity/Fragment 中的生命周期回调中被动被调用的功能，通过监听其生命周期，化被动为主动，成为主动执行相应操作的一方，这就是质的转变。