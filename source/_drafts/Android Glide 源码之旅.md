---
title: Android Glide 源码之旅
tags:
	- android
	- android 源码
	- glide
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
>   一个用来为 Glide 管理和开始请求的类。可以使用 Activity 、Fragment 和其他连接了生命周期的事件来智能地停止、开始或者重新开始一些请求。通过实例化一个实例，或者从 Activity 、 Fragment 的生命周期中已经构建的拿出，然后使用静态方法`Glide.with.load`就 Ok 了。

首先看一下这个类的方法，我们最熟悉的就是`load()`这个方法，通过传入一个图片的 Uri 、Url 或其他路径，让 Glide 知道我们需要得到的图片具体的获取方式。`load()`方法也有多个重载，分别对应的是 Bitmap byte[] Drawable File Integer Object String Uri URL 等。由于我一般都是传入一个图片的链接来加载图片，所以首先看一下`load(String)`方法的实现：

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
```

这个方法的返回值是`RequestBuilder<Drawable>`，然后在方法中调用的是`asDrawable().load()`，那顺着这两个方法，首先找到`asDrawable()`方法的定义：

```java
public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);
}
```

这个方法的作用只是将「 Drawable 」由方法名变为了方法的参数，没有什么值得关注的，那么就直接来看`as()`方法：

```java
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

可以看到，在这个方法里创建了一个 RequestBuilder 的实例，其中将上面的`Drawable.class`还是作为了参数传入了 RequestBuilder 的构造方法。然后再看一下其他的如`load(File)`方法，发现它们也都是调用了`asDrawable().load()`方法，同时这个类还提供了使用者可以主动调用的如`asBitmap()`、`asFile()`等方法，如`asBitmap`的实现：

```java
public RequestBuilder<Bitmap> asBitmap() {
  return as(Bitmap.class).apply(DECODE_TYPE_BITMAP);
}
```

由此可知，RequestManager 类的主要作用就是根据用户的需求构建不同类型的 RequestBuilder ，然后将这个实例返回给用户进行下一步操作。

### RequestBuilder

在 RequestManager 中，调用的`load()`方法最终都会变为调用 RequestBuilder 的`load()`方法，还是首先看一下这个类的介绍：

>    A generic class that can handle setting options and staring loads for generic resource types.
>
>    一个可以为通用的资源类型处理设置选项、开始加载的通用类。

首先看一下`load(String)`方法：

```java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
  return loadGeneric(string);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```

这里只是将图片的链接以参数的方式传入了类内部，所以这个方法还不是加载图片的，他只是先将这个数据保存了下来。至于其他的一些`load()`方法，也都有调用了`loadGeneric()`这个方法，只不过它们有的还会调用`apply()`这个方法，设置一下 RequestOptions 的内容。

调用了`load()`方法之后，就可以直接调用`into()`方法将对应的图片资源加载到 ImageView 等显示图片的控件上了。那么就来看一看`into(ImageView)`方法：

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);
  RequestOptions requestOptions = this.requestOptions;
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }
  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions);
}
```

可以得知，当我们调用`into(ImageView)`方法的时候，它会将 ImageView 通过一定的转换变成 ViewTarget ，再调用`into(Y extends Target, RequestListener, RequestOptions)`,而把一个 ImageView 转换为 ViewTarget 用的是`GlideContext.buildImageViewTarget(ImageView, Class)`方法，这个 Class 就是当时传入的`Bitmap.Class`、`Drawable.class`等。如果当时传入的是`Drawable.class`，最终就会返回一个`DrawableImageViewTarget`类的实例，而在这个类中实现了`setResource`方法：

```java
protected void setResource(@Nullable Drawable resource) {
  view.setImageDrawable(resource);
}
```

图片资源加载完成之后，就是通过这个方法显示到 ImageView 上去的。得到这个 Traget 之后，Glide 会接着调用`into(Target, RequestListener, RequestOptions)`方法：

```java
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @NonNull RequestOptions options) {
  Util.assertMainThread();
  Preconditions.checkNotNull(target);
  if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }
  options = options.autoClone();
  Request request = buildRequest(target, targetListener, options);
  Request previous = target.getRequest();
  if (request.isEquivalentTo(previous)
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();
    // If the request is completed, beginning again will ensure the result is re-delivered,
    // triggering RequestListeners and Targets. If the request is failed, beginning again will
    // restart the request, giving it another chance to complete. If the request is already
    // running, we can let it continue running without interruption.
    if (!Preconditions.checkNotNull(previous).isRunning()) {
      // Use the previous request rather than the new one to allow for optimizations like skipping
      // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
      // that are done in the individual Request.
      previous.begin();
    }
    return target;
  }
  requestManager.clear(target);
  target.setRequest(request);
  requestManager.track(target, request);
  return target;
}
```

一般情况下，调用到`into()`方法之后，图片就会进行加载然后显示到 ImageView 上面去了，这个加载就是由 Request 这个类来完成的。可以看到，倒数第二句是`requestManager.track(Target, Request)`，然后再看这个方法的实现：

```java
// in RequestManager
void track(@NonNull Target<?> target, @NonNull Request request) {
  targetTracker.track(target);
  requestTracker.runRequest(request);
}

// in RequestTracker
public void runRequest(@NonNull Request request) {
  requests.add(request);
  if (!isPaused) {
    request.begin();
  } else {
    pendingRequests.add(request);
  }
}
```

可以看到，这个 Request 最终对进入一集合中去统一管理，同时调用它的`begin()`方法开始发起请求。

所以这里主要涉及到两个类，`ViewTarget`和`Request`，它们分别对应的外部显示图片的控件和内部加载图片的逻辑，下面就先来看看这两个类。

### ViewTarget

ViewTarget 的介绍：

>   A base Target for laoding Bitmap into View that provides default implementations for most most methods and can determine the size of views using a OnDrawableListener.
>
>   一个用来加载 Bitmap 到一个 View 的基础 Target 类，其中这个 View 提供了大多数方法的默认实现，并且可以使用一个 OnDrawableListener 确定 view 的大小。

总而言之，这个类持有了我们需要显示图片的 View ，同时可以在这个类中看到，它将 Request 作为一个 Tag 传入了 View 中，并通过获取这个 Tag 得到 Request 。还能再某种程度上控制 Request 的加载过程，以及添加和解除 View 和 Request 的联系等。

### Request

>   A request that loads a resource for an Target.
>
>   一个用来为 Target 加载资源的请求。

由介绍就可以看出，Request 就是一个用来加载图片资源的请求。但是 Request 本身只是一个接口，并没有对应方法的实现，所以需要转到它的实现类中去看，发现有一个类 SingleRequest 实现了 Request。

在之前的如 RequestBuilder 类中我们可以看到，加载图片时调用了`Request.begin()`方法，在 Request 接口中关于次方法的生命就是：

```java
/**
 * Starts an asynchronous load.
 */
void begin();
```

所以，首先看一下 SingleRequest 类中关于`begin()`方法的实现：

```java
public void begin() {
  assertNotCallingCallbacks();
  stateVerifier.throwIfRecycled();
  startTime = LogTime.getLogTime();
  if (model == null) {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      width = overrideWidth;
      height = overrideHeight;
    }
    // Only log at more verbose log levels if the user has set a fallback drawable, because
    // fallback Drawables indicate the user expects null models occasionally.
    int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
    onLoadFailed(new GlideException("Received null model"), logLevel);
    return;
  }
  if (status == Status.RUNNING) {
    throw new IllegalArgumentException("Cannot restart a running request");
  }
  // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
  // that starts an identical request into the same Target or View), we can simply use the
  // resource and size we retrieved the last time around and skip obtaining a new size, starting a
  // new load etc. This does mean that users who want to restart a load because they expect that
  // the view size has changed will need to explicitly clear the View or Target before starting
  // the new load.
  if (status == Status.COMPLETE) {
    onResourceReady(resource, DataSource.MEMORY_CACHE);
    return;
  }
  // Restarts for requests that are neither complete nor running can be treated as new requests
  // and can run again from the beginning.
  status = Status.WAITING_FOR_SIZE;
  if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    onSizeReady(overrideWidth, overrideHeight);
  } else {
    target.getSize(this);
  }
  if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
      && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
  }
  if (IS_VERBOSE_LOGGABLE) {
    logV("finished run method in " + LogTime.getElapsedMillis(startTime));
  }
}
```

上面就是`begin()`方法的全部代码，对当前的 status 进行了判断，处于不同的状态会给予不同的响应，其中当`status == Status.COMPLETE`的时候，会调用`onResourceReady(Resource, DataSource)`方法，表示当加载成功之后作出的响应，然后进行一系列资源的判断，最后当所有资源请求无误之后，就会调用`Target.setResource()`方法，对应在 DrawableImageViewTarget 的就是为 ImageView 设置图片资源。但是这个资源在最开始肯定是不可能处于`Status.COMPLETE`状态的，所以当第一次调用的时候调用的是`onSizeReady()`这个方法，首先会验证控件的宽高是否合法，如果合法，就调用`onSizeReady()`方法，然后开始调用 Engine 的加载图片的方法。

```java
public void onSizeReady(int width, int height) {
  ...
  ...
  loadStatus = engine.load(
      glideContext,
      model,
      requestOptions.getSignature(),
      this.width,
      this.height,
      requestOptions.getResourceClass(),
      transcodeClass,
      priority,
      requestOptions.getDiskCacheStrategy(),
      requestOptions.getTransformations(),
      requestOptions.isTransformationRequired(),
      requestOptions.isScaleOnlyOrNoTransform(),
      requestOptions.getOptions(),
      requestOptions.isMemoryCacheable(),
      requestOptions.getUseUnlimitedSourceGeneratorsPool(),
      requestOptions.getUseAnimationPool(),
      requestOptions.getOnlyRetrieveFromCache(),
      this);
  // This is a hack that's only useful for testing right now where loads complete synchronously
  // even though under any executor running on any thread but the main thread, the load would
  // have completed asynchronously.
  if (status != Status.RUNNING) {
    loadStatus = null;
  }
  if (IS_VERBOSE_LOGGABLE) {
    logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
  }
}
```

如上代码中的`load()`方法，就是 Engine 用来加载图片的，同时将加载结果返回给当前状态。

### Engine

>   Responsible for starting loads and managing active and cached resources.
>
>   负责开始加载并管理可用的和已经缓存的资源。

在 Request 类中，通过调用 Engine 的`load()`方法来加载图片资源，所以现在先看这个方法：

```java
public <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb) {
  Util.assertMainThread();
  long startTime = LogTime.getLogTime();
  EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
      resourceClass, transcodeClass, options);
  EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
  if (active != null) {
    cb.onResourceReady(active, DataSource.MEMORY_CACHE);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
  }
  EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
  if (cached != null) {
    cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return null;
  }
  EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
  if (current != null) {
    current.addCallback(cb);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Added to existing load", startTime, key);
    }
    return new LoadStatus(cb, current);
  }
  EngineJob<R> engineJob =
      engineJobFactory.build(
          key,
          isMemoryCacheable,
          useUnlimitedSourceExecutorPool,
          useAnimationPool,
          onlyRetrieveFromCache);
  DecodeJob<R> decodeJob =
      decodeJobFactory.build(
          glideContext,
          model,
          key,
          signature,
          width,
          height,
          resourceClass,
          transcodeClass,
          priority,
          diskCacheStrategy,
          transformations,
          isTransformationRequired,
          isScaleOnlyOrNoTransform,
          onlyRetrieveFromCache,
          options,
          engineJob);
  jobs.put(key, engineJob);
  engineJob.addCallback(cb);
  engineJob.start(decodeJob);
  if (Log.isLoggable(TAG, Log.VERBOSE)) {
    logWithTimeAndKey("Started new load", startTime, key);
  }
  return new LoadStatus(cb, engineJob);
}
```

这个方法的基本逻辑就是，先根据当前要获取的资源构建一个 EngineKey 实例，用来作为从已经加载得到的可用的或缓存的资源中得到当前请求资源的键，然后首先从活跃的资源表中查找是否有对应着这个 key 的资源，使用`loadFromActiveResources()`方法，如果没有找到，那就到缓存中寻找，使用`loadFromCache()`方法，如果再没有找到，再到工作列表中寻找，如果正在工作，就返回工作状态，如果再找不到，那就只能创建一个工作并加到工作列表，然后开始这项工作。

### EngineJob

>   A class that manages a load by adding and removing callbacks for for the load and notifying callbacks when the load completes.
>
>   一个通过添加和移除资源加载的回调来管理资源并在资源加载完成之后通知回调方法的类。

这个方法实现了一些关于资源加载完成之后的回调方法，如`onResourceReady()` `onLoadFail()` 等方法，使用了 Handle-Message 的方式处理线程之间的信息传递，然后就是使用 GlideExecutor 执行加载工作。

### DecodeJob

而这个要执行的工作就是 DecodeJob，先看一下 DecodeJob 的介绍：

>   A class responsible for decoding resources either from cached data or from the original source and applying transformations and transcodes.
>
>   一个负责解码从缓存或者源地址得到的资源并且进行一些转化的类。

由于这个类是 Runable 的实现，同时也是把这个类当作一个作业送入 ThreadPoolExecutor 中的，所以这个类的入口应该是`run()`方法。

```java
public void run() {
  // This should be much more fine grained, but since Java's thread pool implementation silently
  // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
  // that something is failing.
  TraceCompat.beginSection("DecodeJob#run");
  // Methods in the try statement can invalidate currentFetcher, so set a local variable here to
  // ensure that the fetcher is cleaned up either way.
  DataFetcher<?> localFetcher = currentFetcher;
  try {
    if (isCancelled) {
      notifyFailed();
      return;
    }
    runWrapped();
  } catch (Throwable t) {
    // Catch Throwable and not Exception to handle OOMs. Throwables are swallowed by our
    // usage of .submit() in GlideExecutor so we're not silently hiding crashes by doing this. We
    // are however ensuring that our callbacks are always notified when a load fails. Without this
    // notification, uncaught throwables never notify the corresponding callbacks, which can cause
    // loads to silently hang forever, a case that's especially bad for users using Futures on
    // background threads.
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "DecodeJob threw unexpectedly"
          + ", isCancelled: " + isCancelled
          + ", stage: " + stage, t);
    }
    // When we're encoding we've already notified our callback and it isn't safe to do so again.
    if (stage != Stage.ENCODE) {
      throwables.add(t);
      notifyFailed();
    }
    if (!isCancelled) {
      throw t;
    }
  } finally {
    // Keeping track of the fetcher here and calling cleanup is excessively paranoid, we call
    // close in all cases anyway.
    if (localFetcher != null) {
      localFetcher.cleanup();
    }
    TraceCompat.endSection();
  }
}
```

可以看到，这里还是首先做了一些资源与操作正确性的判断，然后调用 了`runWrapper()`方法，这里需要关注到的是 DataFetcherGenarator 类，顾名思义，就是用来构建真正的 DataFetcher 的类，