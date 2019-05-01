---
title: ViewModel 简介
date: 2019-05-01 23:00
tags:
	- android
---

ViewModel ，开发中的又一大利器，其名字也正好对应着 MVVM 中 ViewModel ，这并不是巧合。从功能上看，它负责为 Activity 和 Fragment 提供和管理数据，也就是为 View 层提供数据服务，它能够在屏幕方向切换的时候保存数据，也能够在多个 Fragment 组件中进行通信，想拥有此功能必然有一定的基础支撑，首先可以确定的是，ViewModel 需要比 Activity 的生命周期更长，那么需要有集中存储 ViewModel 的地方，还需要能够根据 Activity 得到对应的 ViewModel 。然后就是二者的通信手段，由于生命周期不同步，ViewModel 就不应该持有 Activity 的引用，而之前也正说过 LiveData ，可以由 LiveData 负责完成与 Activity 的通信，ViewModel 只需要存储这些 LiveData 即可，虽然 LiveData 与 Activity 生命周期不一致，但是它提供的有关于生命周期的管理，而实际上，它们俩也确实就是需要在一起使用的。

具体的如何使用先不谈，先看 ViewModel 本身。上面也说到了 ViewModel 的一个特性，当然这些特性是需要在代码层面予以支持的。了解如何使用肯定很快，但了解其实现原理及思想，则需要从源码开始，条分缕析。

首先看 ViewModel 类，它是一个抽象类，没有抽象方法。在其内部有一个 ConcurrentHashMap ，可用于多线程中的数据存储，通过`setTagIfAbsent()`和`getTag()`存取数据，不过在使用的时候更多的还是给它添加 LiveData 成员变量。要想使用 ViewModel 必须先要实现一个子类，再说 ViewModel 的获取，当然可以直接在 Activity 中调用其构造方法实例化一个对象，但是这种时候的 ViewModel 没有太多意义，上面已经说过 ViewModel 可以用于数据恢复，所以必然 ViewModel 不能仅仅是 Activity 的一个成员变量。

### 获取 ViewModelStore

一般情况下获取 ViewModel 需要使用到类 ViewModelProviders ，常用的代码为`ViewModelProviders.of().get()`，便可得到一个 ViewModel 实例，`of()`方法有四个重载，分别是：

```java
public static ViewModelProvider of(Fragment);
public static ViewModelProvider of(Fragment, Factory);
public static ViewModelProvider of(FragmentActivity);
public static ViewModelProvider of(FragmentActivity, Factory);
```

其中第1、3方法会转而调用第2、4方法，这个方法所需要的参数有三个，Fragment/FragmentActivity 和 Factory ，返回的是 ViewModelProvider 实例，以其中一个为例：

```java
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(activity.getViewModelStore(), factory);
}
```

可见第一个参数的作用就是获取 ViewModelStore ，然后用于实例化 ViewModelProvider 。ViewModelStore 有两种获取方式，分别在 FragmentActivity 和 Fragment 中。先看在 Activity 中是何如获取的。

上面的`getViewModelStore()`方法在 ComponentActivity 有实现，如下：

```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```

ViewModelStore 有两种来源，一种是直接创建，一种是从 NonConfigurationInstances 中获取，当一个 Activity 是因为屏幕方向改变而销毁并重新创建的时候，可以通过`getLastNonConfigurationInstance()`方法得到这个对象，对象里面就含有上一个 Activity 的 ViewModelStore ，具体的过程中涉及到 Activity 的销毁启动等流程，不作赘述。

从 Activity 中得到 ViewModelStore 的过程就是这样，下面看看从 Fragment 中取得 ViewModelStore 的过程。首先也是从`of()`方法开始，调用了 Fragment 的`getViewModelStore()`方法，接着又转交给`FragmentManagerImpl.getViewModelStore()`，最终转交给`FragmentManagerViewModel.getViewModelStore()`，总之调用了好几重才真正有一些现实功能。一步一步看，首先调用的是 FragmentManager ，这个 FragmentManager 就是在 Activity 中负责管理 Fragment 的 FragmentManager ，其初始化是在 FragmentActivity 中，当一个 Fragment 被加入到 Activity 中时这个 FragmentManager 就会被传给 Fragment ，这涉及到 Fragment 的添加流程，不多说。

然后在 FragmentManager 中又调用了 FragmentManagerViewModel 的`getViewModelStore()`方法，这个类是干什么的？从名字上看它是一个 ViewModel ，应该是继承于 ViewModel 的，它在 FragmentManager 初始化的时候生成，代码如下：

```java
public void attachController(@NonNull FragmentHostCallback host,
                             @NonNull FragmentContainer container, @Nullable Fragment parent) {
    if (mHost != null) throw new IllegalStateException("Already attached");
    mHost = host;
    mContainer = container;
    mParent = parent;
    if (parent != null) {
        mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
    } else if (host instanceof ViewModelStoreOwner) {
        ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
        mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
    } else {
        mNonConfig = new FragmentManagerViewModel(false);
    }
}
```

有三种情况，如果这是一个 Activity 的 FragmentManager ，则会进入第二种情况，先取得 Activity 的 ViewModelStore ，然后利用这个参数创建 FragmentManagerViewModel 实例。

```java
static FragmentManagerViewModel getInstance(ViewModelStore viewModelStore) {
    ViewModelProvider viewModelProvider = new ViewModelProvider(viewModelStore,
            FACTORY);
    return viewModelProvider.get(FragmentManagerViewModel.class);
}
```

如上，这里并没有使用 ViewModelProviders 获取实例，而是直接创建的 ViewModelProvider 实例，这是当然，这个过程本身就是为了用户级 Fragment 中获取 ViewModel 实例存在的，如果再利用这么一个流程，就进入死循环了。还有，这里之所以可以直接创建 ViewModelProvider ，是因为 FragmentManagerViewModel 虽然每次都创建新的对象，但是在这里已经取得了 ViewModelStore ，这个 ViewModelStore 与 Activity 的生命周期保持一致（甚至能长于 Activity 的生命周期，当 Activity 因为屏幕方向改变销毁重建时），所以可以保证重用。

那么 FragmentManagerViewModel 有什么功能？首先明确，它是与 FragmentManager 对应的。然后，在他的内部有一个 Set 和两个 HashMap ，

```java
private final HashSet<Fragment> mRetainedFragments = new HashSet<>();
private final HashMap<String, FragmentManagerViewModel> mChildNonConfigs = new HashMap<>();
private final HashMap<String, ViewModelStore> mViewModelStores = new HashMap<>();
```

单从各变量名就可以看出它们保存了那些东西，第一个保存的是保留 Fragment ，用于在 Activity 销毁的时候保存应该继续保留的 Fragment ，第二个是用于保存子 FragmentManagerViewModel 的，每一个 Fragment 都有用于维护嵌套 Fragment 的 FragmengManager ，当然也有对应的 FragmentManagerViewModel ，第三个变量，用于保存各 Fragment 的 ViewModelStore ，

```java
ViewModelStore getViewModelStore(@NonNull Fragment f) {
    ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
    if (viewModelStore == null) {
        viewModelStore = new ViewModelStore();
        mViewModelStores.put(f.mWho, viewModelStore);
    }
    return viewModelStore;
}
```

初始时 mViewModelStores 为空，在之后的`getViewModelStore()`中，每一个 Fragment 都有一个对应的 ViewModelStore ，每获取到一个就会将其加入 mViewModelStores ，以便下次重用，所以在这里可以确定的是，在同一个 Activity 中不同的 Fragment 有不同的 ViewModelStore ，在不同的 Activity 下，所有的 Fragment 都会有不同的 ViewModelStore 。

以上就是 Fragment 获取 ViewModelStore 的过程。

### 获取 ViewModel

再回过头看 ViewModelProviders ，它会利用这个可重用的 ViewModelStore 构建 ViewModelProvider 实例，然后就可以调用`get()`方法获取 ViewModel 实例。

```java
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        viewModel = (mFactory).create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

先判断，匿名类或内部类不能作为 ViewModel 使用。然后从 ViewModelStore 中寻找是否有可复用实例，没有的话就会就地取材构建，构建需要使用到 Factory ，Factory 有两种获取途径，开发者自行编写一个 Factory ，或使用默认的。默认的 Factory 在 ViewModelProviders 中构建，构建出的是 AndroidViewModelFactory 实例，其创建方法如下：

```java
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
        //noinspection TryWithIdenticalCatches
        try {
            return modelClass.getConstructor(Application.class).newInstance(mApplication);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
    return super.create(modelClass);
}
```

只能用于一种情况，那就是 ViewModel 的构造方法只有一个参数且参数为 Application 的时候才能使用，否则就会抛出运行时异常。所以大部分情况下，应该都还是使用自定义的 Factory ，

```java
public interface Factory {
    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```

Factory 只有一个抽象方法，它有两个抽象子类，NewInstanceFactory 和 KeyedFactory ，执行到最后一步，调用`create()`方法就可以创建出最后的 ViewModel 实例了，到此也终于获得了这个 ViewModel 。

从以上的流程就可以看出，当 Activity 因为屏幕旋转而重建的时候，ViewModel 可以被复用，而 ViewModel 用于保存 Activity/Fragment 所需的数据，也就是说能够保障数据不会因为重建而销毁，也不再需要再在`onSaveInstanceState()`和`onCreate()`中做大量的数据保存与数据恢复相关的操作了。

之前也提到 ViewModel 可以用于 Fragment 之间、Fragment 与 Activity 通信，其原理就是利用 ViewModel 的复用机制，可以在 Fragment 中调用`getActivity()`获取到 Activity 实例，进而获取与 Activity 相关的 ViewModel ，从而使得 Activity 与 Fragment 共用一个 ViewModel ，从而利用 ViewModel 作为中介达到数据交换的目的，Fragment 之间的通信也是如此，多个 Fragment 共用同一个 ViewModel 进行数据交换。

### ViewModel 使用

ViewModel 可以重用，它的生命周期比 Activity 长，但是在数据交换方面，大致有两种方法，即 MVP 与 MVVM 两种对应的方式，回调接口或利用观察者模式。先说回调，要通过回调完成交换，首先 ViewModel 需要有 Activity 的引用，但是因为生命周期的不一致，持有引用的时候就需要额外小心，必须要在对应的生命周期中释放引用，还要能及时持有新的 Activity 的引用，这还是比较复杂的。

所以 ViewModel 的选择是，不直接与 Activity 建立联系，而是使用 LiveData 保存数据，然后利用 LiveData 能够与 Activity/Fragment 建立联系并能自动监听其生命周期的特性，很好的解决了二者生命周期不同步的问题，此时，ViewModel 作为数据的提供者，LiveData 作为数据，在 Activity 中监听 LiveData 的改变，很好的完成了从 ViewModel 到 Activity 数据的传送问题，至于从 Activity 到 ViewModel ，因为生命周期没有 ViewModel 长，Activity 则可以直接持有 ViewModel 的引用，调用 ViewModel 提供的方法，完成这一方向的数据传送。

如此，一个 MVVM 的模型就形成了。