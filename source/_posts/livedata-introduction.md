---
title: LiveData 简介
date: 2019-04-30 22:00
tags:
	- android
---

LiveData 是一个用于存储数据的类，但它并不是仅仅用于数据的存储，还用于传递，每一个 LiveData 都是一个观察者模式的实现。其存储功能即在其内部有一个 mData 变量，并提供了`getValue()`和`setValue()`等方法对这个数据进行操作，其传递功能，就在于这个类也是观察者模式中的发布者，它提供了`observe()`方法为其增加观察者，这个观察者观察的内容，就是 LiveData 中的数据，当改变 LiveData 中存储的数据时，它就会将这个改变发布给所有的观察者，让观察者根据这个值的改变作出相应改变，不止如此，LiveData 还于 Lifecycle 建立了联系，在添加观察者的时候，还需要与之相联系的 LifecycleOwner ，生命周期提供者，也就是 Activity 或 Fragment 。有了生命周期加持之后，LiveData 就不仅仅是一个数据传递者，而是一个有血有肉的数据管理者，它可以根据当前的状态决定是否传递数据。

下面看看 LiveData 的具体实现。

### 观察者角度

从观察者的角度来看，首先也是唯一需要的方法就是`observe()`，它有两个参数，LifecycleOwner 和 Observer ，两个都是接口，前者一般是 Activity/Fragment ，后者则是开发者自己实现的逻辑。

```java
public interface Observer<T> {
    /**
     * Called when the data is changed.
     * @param t  The new data
     */
    void onChanged(T t);
}
```

只有一个`onChanged()`方法用于通知观察者数据的更改。

然后看`observe()`的具体实现：

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}

class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;
    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }
    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }
    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }
    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

首先判断线程，然后将两个参数包装成 LifecycleBoundObserver 对象，并将其加入 mObservers 。包装类 LifecycleBoundObserver ，继承自 ObserverWrapper ，实现了 LifecycleEventObserver 接口，在之前关于 Lifecycle 的文章里讲过，它在 Lifecycle 机制中作为观察者，其回调方法是`onStateChanged()`，在这个方法中可以看到，LifecycleBoundObserver 根据当前的状态，选择将 LiveData 的观察者移除，或者是改变这个观察者状态（是否处于活跃状态）。

观察者的订阅过程就这么多，主要就是将观察者保存起来。再看`observe()`对参数，虽然有 LifecycleOwner ，但是 LiveData 本身的运作并不受生命周期的影响，而是在类 LifecycleBoundObserver 里面，将 LifecycleOwner 和 这个观察者之间建立了一种联系，通过生命周期判断这个观察者是否应该处于活跃状态，以及是否需要将其移除。而对于 LiveData 本身来说，它的生命周期与 LifecycleOwner 的生命周期无关，它主要做的，就是在数据改变的时候将这个改变事件发布给所有的观察者。

不过除此之外，LiveData 还提供了一种不需要 LifecycleOwner 的方式，`observeForever()`，这就更像是一个纯粹的观察者模式的实现，没有了生命周期的束缚，观察者就不会自动移除，只能调用`removeObserver()`。

### 发布者角度

从发布者角度说的话，LiveData 提供了`setValue()`和`postValue()`两个方法用于更改数据，二者的区别主要是线程的问题，因为 LiveData 默认的只能工作在主线程，上面的`observe()`方法中也有关于线程的检测，所以如果是需要在子线程中发布结果的话，需要先转换一下线程，`postValue()`中就有线程转换的操作，切换完线程，最终还是调用`setValue()`完成数据的改变的。

```java
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

逻辑比较简单，就是给 mData 赋值，然后调用`dispatchingValue()`方法将这个更新发布给观察者，然后还有就是这个 mVersion ，指的是 mData 的值更新的次数，这个是在后面分发的时候需要使用到的，判断观察者现有的值，是否是最新的值。

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

在`dispatchingValue()`方法中有一个二重循环，内部的循环就是遍历所有的观察者，调用`considerNotify()`告知其更新数据，至于外层循环，是一个关于 mDispatchInvalidated 和 mDispatchingValue 的循环，按照代码所显示的，这是一个控制多线程同时访问的功能，当多个线程同时调用了此方法请求分发更新的时候，如果一个线程中正在分发，则另一个线程就不会再进入这个分发的环节，而是要求那个正在分发的线程重新分发。

那么此处有一个问题，上面说到过，`setValue()`是需要运行在主线程的，如果是这样，那就不存在多线程环境，何来的这样一种情况？寻找调用`dispatchingValue()`的地方，发现有两个，一个就是`setValue()`，另一个是在 ObserverWrapper 的`activateStateChanged()`中，貌似这里并没有规定要在主线程中执行，但是这个方法是在`onStateChanged()` 和 `observeForevent()`中调用的，后者也确定是运行在主线程，至于前者，究其根源，`onStateChanged()`还是在 Activity/Fragment 的生命周期回调方法中调用的，此处的代码一定运行在主线程，那么按图索骥，可以初步判断`activateStateChanged()`应该也是执行在主线程的，那么在`dispatchingValue()`方法中这个关于多线程的处理就显得有些难以理解。

不过再回过头来看这个处理过程，与其说它是一个多线程同步的处理，它更像是一个避免重入的处理。抛开多线程的情况之后，有可能在多个`dispatchingValue()`同时调用的情况就是重入，即在一个`dispatchingValue()`的执行过程中又调用了`dispatchingValue()`方法，这种情况有可能吗？肯定是有可能的，这个方法的功能就是分发新的值到观察者，调用了`considerNotify()`方法，在这个方法里又会调用 Observer 的`onChanged()`方法，那么有一种情况很有可能出现，那就是在一个值更新的操作里，会接着触发数据的更新，然后经过一系列的调用最后又会调用到`dispatchingValue()`，也就是说在这个方法里调用了这个方法，也就是重入，为了避免重入，就可以采用这种方式。

那么这样的处理是为了什么？就是为了避免重入，有可能出现一种情况，那就是在一个`dispatchingValue()`调用的过程中，又会大量地调用这个方法，这种多重方法的调用则会大量消耗内存，并且，这还是一个递归过程，意味着当新的`dispatchingValue()`调用完了之后还会调用上一层`dispatchingValue()`，但是此时的最新的值已经被发布过了，再继续执行这个方法也没有什么意义了。所以，为了避免以上两种情况，作者在编写这个方法的时候，利用一个二重循环解决了重入的问题，思想确实巧妙。

下一步就是调用`considerNotify()`方法，判断观察者是否存活，以及最新的值是否被分发以决定是否执行分发操作`onChanged()`，之后就会进入开发者实现的逻辑中去。

### 子类

LiveData 是一个抽象类，虽然它好像并没有抽象方法，所以使用 LiveData 时需要使用它的子类 MutableLiveData ，这个类并没有对 LiveData 功能上的扩展，只是将 LiveData 中被 protected 修饰的`getValue()`和`setValue()`方法，改成了 public ，这时这两个方法才能被外界使用。

MutableLiveData 也有一个子类，MediatorLiveData ，相比于 MutableLiveData 增加了 mSources 这个东西，以及对应的方法`addSource()`和`removeSource()`，其参数为 LiveData 和 Obsercver ，可以看出这也是一个观察者，只不过这里是将 MediatorLiveData 作为观察者，观察另一个 LiveData 的数据变化，其使用场景，就是当一个值需要跟随另一个值改变的时候，比如有两个 LiveData ，分别存储用户 id 和用户名，那么用户名需要跟随 id 的改变而改变，此时就可以将用户名对应的 LiveData 使用 MediatorLiveData 实例化，并调用`addSource()`方法，监听另一个 LiveData 即 id 的变化，当 id 发生变化时立马根据 id 找到新的用户并求出新用户的用户名。

当然这种功能也可以在开发者层面实现，先在 Acitivity 中增加一个 id LiveData 的监听，监听到改变之后，再去调用某一个方法请求用户名 LiveData 作出改变，二者本质上没有太大区别，只不过实现的方式不太一样，不过要从思想上看的话，使用了 MediatorLiveData 的数据是主动监听 id 的变化而主动更新，后者则是由 Activitiy 收到 id 更新之后请求用户名更新，这则是一个被动的过程。

### 总结

LiveData 作为数据存储的工具，关于其使用，一般只需要了解`observe()`、`setValue()`、`postValue()`即可。

然后从实现的角度来看 LiveData ，它是观察者模式的实现，与其他许多工具（本地广播、EventBus等）有着类似的实现方法，不过它的特色就在于与 Lifecycle 的结合，使得观察者只需要主动执行添加操作，而不需要关心什么时候该注销观察者，在其内部有一个自成体系的管理机制，是其他工具所不具有的。也正是这种特点，使得它可以与 ViewModel 一起组建 MVVM 架构。