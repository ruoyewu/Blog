---
title: Android Glide 源码之旅
tags:
	- android
	- android 源码
---

## Glide 介绍

Glide 是 Android 平台上一个功能非常丰富的图片加载库，能够通过多种方式加载本地、网络上的图片，将其转变为`Drawable`或`Bitmap`或者是知道将图片显示到指定的`ImageView`中去，同时还支持磁盘缓存、内存缓存及混合使用缓存等多种方式。对于一个 Android 应用来说，图片的加载与显示肯定是不可或缺的一部分，所以，很有必要熟悉至少一种图片加载库来提升自己的开发效率。

使用过 Glide 的人最熟悉的一句代码大概就是`Glide.with(context).load("https://wuruoye.com/images/avatar.jpg").into(imageView)`了，所以现在就首先跟踪这样一个足迹看一看 Glide 到底是怎么样一步一步将这样网络图片加载到这个 ImageView 中的。

## `Glide.with().load().into()`

### Glide

首先转到 Gilde 这个类中，看到类的介绍为：

>   A singleton to present a simple static interface for building requests with RequestBuilder and maintaining an Engine, BitmapPool, DiskCache and MemoryCache.
>
>   一个用来提供简单的静态接口的单例，它使用 RequestBuilder 来创建请求，并且维护 Engine、BitmapPool、DishCache 和 MemoryCache。

总体意思就是说，这就是我们使用 Glide 的时候的一个接口，我们通过这个类进入进入这个用来完成复杂工作的 Glide 系列里面。

首先就会看到 Glide 中提供了多种重载的`with()`方法，但是他们最终都会执行`getRetriever(Context).get(Context)`方法，这个方法中主要就是对传入的`context`进行非空检测，然后就会看到，在`getRetriever(Context)`方法里面又会看到它调用了`Glide.get(Context)`方法，`get(Context)`方法的定义如下：

```java
@NonNull
public static Glide get(@NonNull Context context) {
  if (glide == null) {
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context);
      }
    }
  }
  return glide;
}
```

可以看到，这是一个典型的单例模式中创建单例的方法，另外可以看到，这里调用了`checkAndInitializeGlide(Context)`方法创建这个单例。而在这个方法中，

```java
private static void checkAndInitializeGlide(@NonNull Context context) {
  // In the thread running initGlide(), one or more classes may call Glide.get(context).
  // Without this check, those calls could trigger infinite recursion.
  if (isInitializing) {
    throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
        + " use the provided Glide instance instead");
  }
  isInitializing = true;
  initializeGlide(context);
  isInitializing = false;
}
```

它又对并发做了一定的限制。所以真正的创建这个单例的代码还不在这里，它在`initializeGlide(Context)`中。但是再去看`initializeGlide(Context)`的代码，发现它仍旧没有真正实例化这个 Glide 的代码，它只是在原先的基础上调用了`initializeGlide(Context, GlideBuilder)`：

```java
private static void initializeGlide(@NonNull Context context) {
  initializeGlide(context, new GlideBuilder());
}
```

这里用到了一个没有内容的 GlideBuilder 参与 Glide 实例的初始化。再转到`initializeGlide(Cotnext, GlideBuilder)`，就可以看到，真正的 Glide 的单例初始化的地方在这里。这个方法主要做的工作就是，将一些预设的配置信息加载出来，参与 Glide 的初始化。代码如下：

```java
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
  Context applicationContext = context.getApplicationContext();
  GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
  List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
  ...
  ...
  for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.applyOptions(applicationContext, builder);
  }
  if (annotationGeneratedModule != null) {
    annotationGeneratedModule.applyOptions(applicationContext, builder);
  }
  Glide glide = builder.build(applicationContext);
  for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.registerComponents(applicationContext, glide, glide.registry);
  }
  if (annotationGeneratedModule != null) {
    annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
  }
  applicationContext.registerComponentCallbacks(glide);
  Glide.glide = glide;
}
```

由此可以看到，初始化 Glide 的时候主要用到了两个类，`GeneratedAppGlideMoudle`和`GlideMoudle`，它们的实例分别是`annotationGeneratedMoudle`和`manifestMoudles`，后者是`GlideMoudle`的列表，而由他们各自的名字就可以看出，`annotationGeneratorMoudle`主要是用来加载代码中添加的配置信息，而`manifestMoudles`则是用来加载定义在`AndroidManifest`中的配置信息，加载完成之后通过`applyOptions`将这些配置信息送到`GlideBuider`里面整合起来，最后通过`GlideBuilder.build(Context)`方法得到一个 Glide 实例，到这里为止，这个 Glide 的单例终于是构建出来了，构建出来之后，还要对 Glide 做一些初始化，使用`registerComponents`方法。

###GlideMoudle & AppGlideMoudle

由上面的 Glide 的初始化代码可知，Glide 的初始化主要用到的两个类就是`GeneratedAppGlideMoudle`和`GlideMoudle`，它们都实现了`RegisterComponents`和`AppliesOptions`这两个接口，所以才能使用这两个接口提供的方法来初始化 Glide ，那么这个所谓的`GlideMoudle`和`AppGlideMoudle`到底是用来干什么的，首先看一下`GlideMoudle`的介绍：

>   An interface allowing lazy configuration of Glide including setting options using GlideBuilder and registering ModelLoaders.
>
>   一个允许使用 GlideBuilder 设置选项并注册 ModelLoaders 的 Glide 的惰性配置的接口。

主要的关键字是「 lazy configuration 」，即可以通过它来加载后期的关于 Glide 的配置。

另外看一下`AppGlideMoudle`的介绍：

>   Defines a set of dependencies and options to use when initializing Glide within an application.
>
>   定义了一个在应用中初始化 Glide 时需要用到的依赖和选项集。

### RequestManagerRetriever

最开始讲到，使用`Glide.with()`方法的时候，首先要得到 Glide 的单例，然后调用 Glide 的`getRequestManagerRetriever`得到 RequestManagerRetriever ，那么现在就进入这个类看看它提供了哪些方法。

首先看看介绍：

>   A collection of static methods for creating new RequestManger or retrieving exesting ones from activities and fragment.
>
>   这是一个静态方法的集合，它的功能是用来创建或者从 Activity 或 Fragment 中取回一个已经存在了的 RequestManager。

这个类里面主要的就是通过`get()`方法得到 RequestManager 的实例，现在就先来看一下`get(Activity)`这个方法：

```java
public RequestManager get(@NonNull Activity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    android.app.FragmentManager fm = activity.getFragmentManager();
    return fragmentGet(activity, fm, null /*parentHint*/);
  }
}
```

可以看到，这个方法最终会根据不同的条件分别调用`get(Context)`和`fragmentGet(Activity, FragmentManager, Fragment)`，然后在这两个方法里面也还会对各种条件进行判断，经过一系列的跳转，可以看到，如果是当前线程不是 UI 线程，就会调用到`getApplicationManager(Context)`这个方法来获取 RequestMangaer 实例，代码如下：

```java
private RequestManager getApplicationManager(@NonNull Context context) {
  // Either an application context or we're on a background thread.
  if (applicationManager == null) {
    synchronized (this) {
      if (applicationManager == null) {
        // Normally pause/resume is taken care of by the fragment we add to the fragment or
        // activity. However, in this case since the manager attached to the application will not
        // receive lifecycle events, we must force the manager to start resumed using
        // ApplicationLifecycle.
        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context.getApplicationContext());
        applicationManager =
            factory.build(
                glide,
                new ApplicationLifecycle(),
                new EmptyRequestManagerTreeNode(),
                context.getApplicationContext());
      }
    }
  }
  return applicationManager;
}
```

如果是在 UI 线程里面调用的，最终会进入`fragmentGet()`方法：

```java
private RequestManager fragmentGet(@NonNull Context context,
    @NonNull android.app.FragmentManager fm,
    @Nullable android.app.Fragment parentHint) {
  RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);
  RequestManager requestManager = current.getRequestManager();
  if (requestManager == null) {
    // TODO(b/27524013): Factor out this Glide.get() call.
    Glide glide = Glide.get(context);
    requestManager =
        factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
    current.setRequestManager(requestManager);
  }
  return requestManager;
}
```

可以看到，这是两种获得 RequestManager 的途径，上一种方法是直接通过 RequestManagerFactory 方法来直接创建一个新的 RequestManager ，而第二种则是先检测一下是否已经有保存过的 RequestManager ，如果有，直接拿来用就是了，如果没有，那就创建一个。

### RequestManager

在 RequestManagerRetriever 中得到 RequestManager 之后，就进入了 RequestManger 的逻辑范畴了，首先看这个类的介绍：

>   A class for managing and starting requests for Glide. Can use activity, fragment and connectivity lifecycle events to intelligently stop, start, and restart requests. Retrieve either by instantiating a new object, or to take advantage built in Activity and Fragment lifecycle handling, use the static Glide.load methods with your Fragment or Activity.
>
>   一个用来为 Glide 管理和开始请求的类。可以使用 Activity 、Fragment 和其他连接了生命周期的事件来智能地停止、开始或者重新开始一些请求。

