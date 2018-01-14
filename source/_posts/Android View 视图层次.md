---
title: Android view 视图层次
date: 2018-01-10 22:47
categories: android
tags: 
	- android
	- android 源码
---

## Android 页面显示层次

![](http://blog-1251826226.coscd.myqcloud.com/1976991-810bf9f3f47b221b.png)

由此图可以看到 Activity 作为显示一个页面最基本的元素，它本身并不负责显示页面，而是通过 PhoneWindow 这个窗口映射到显示屏上一系列的 View 来显示内容，PhoneWindow 里面又包含一个 ViewGroup ，即 DecorView ，DecorView 继承自 FragmeLayout ，作为一个页面中最顶层的 View ，它包含多个部分，顶部状态栏、底部导航栏和中间的显示内容的 mContetParent ，我们一般在 Activity 中 通过 setContentView 方法来将一个布局资源文件显示在 Activity 中的时候，一般都是把这个布局加入到 mContentParent 中，mContentParent 对应的 id 是 R.id.content 。

所以要想理解 android 显示页面的时候的布局层次，需要知道以下几个东西：

1.  Activity
2.  PhoneWindow
3.  DocerView
4.  mContentParent
5.  layoutResource

## setContentView

在 Android 中要想显示一个 页面 ，需要以下一些步骤：

1.  写一个类如 MainActivity 继承 Activity（一般情况下会继承 Activity 的子类 AppCompatActivity ）
2.  在 Manifests 文件中声明这个 Activity
3.  在 Activity 中调用 setContentView 方法，将需要显示的 view 与 activity 联系到一起

这时当我们打开 MainActivity 的时候就可以显示出来这个 view 的内容了。setContentView 这个方法有3个重载。在 Activity.java 中这三个类的定义如下：

```java
public void setContentView(@LayoutRes int layoutResId) {
  	getWindow().setContentView(layoutResId);
  	initWindowDecorActionBar();
}

public void setContentView(View view) {
  	getWindow().setContentView(view);
  	initWindowDecorActionBar();
}

public void setContentView(View view, ViewGroup.LayoutParams params) {
    getWindow().setContentView(view, params);
  	initWindowDecorActionBar();
}
```

可以看到，activity 中的 setContentView 都调用了 getWindow() 的 setContentView 方法，而可以看到 getWindow 的定义：

```java
public Window getWindow() {
  	return mWindow;
}
```

同时可以看到 mWindow 初始化的地方在 attach 方法：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window);
  
  	...
}
```

由此便可以看到，这里的 mWindow 其实就是一个 PhoneWindow ，最后再找到 PhoneWindow 中的 setContentView 方法，就能看到我们一直以来在 activity 中调用的 setContentVIew 方法到底是怎么样一个流程：

```java
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

里面有一个 mContentParent ，查看定义为：

```java
// This is the view in which the window contents are placed. It is either
// mDecor itself, or a child of mDecor where the contents go.
ViewGroup mContentParent;
```

可以看到，这里是窗口内容的所在，它是 mDecor 本身或者是 mDecor 的一个子 View ，DecorView 是页面的顶级 View ，包含屏幕里面顶部的状态栏、底部导航栏以及中间显示页面的部分。

其中主要与 layoutResId 相关的是这一句：

```java
if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
    final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
            getContext());
    transitionTo(newScene);
} else {
    mLayoutInflater.inflate(layoutResID, mContentParent);
}
```

可以看到，这个方法里面是将我们所要展示的页面的 view 添加到了 mContentParent 里面去，而关于 mContentParent ，可以看到在 setContentView 中首先对 mContentParent 是否为空作了判断，如果为空，则调用 installDecor 方法对其以及 mDecor 作初始化，否则就直接清空 mContentParent 然后加入当前布局页面的 view。而初始化的方法 installDecor 具体代码为：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeOptionalFitsSystemWindows();
        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);
      	...
    }
  	...
}
```

方法内部首先对 mDecor 作了初始化，然后对 mContentParent 初始化，使用 generateLayout 方法，所以再看 generateLayout 方法：

```java
protected ViewGroup generateLayout(DecorView decor) {
  	...
    // Inflate the window decor.
    int layoutResource;
  	int features = getLocalFeatures();
  
  	//一系列操作使不同的配置获得到不同的 layoutResource
  	...
    mDecor.startChanging();
	mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
  	...
    return contentParent;
}
```

由此可以看到，mContentParent 对应的 view 就是那个 id 为 R.id.content 的 view，同时还根据 features 得到了 mDecor 对应的布局资源文件 layoutResource ，将 mDecor 与相应的布局页面联系起来。

所以 installDecor 方法就是做一定的工作来初始化 mDecor 和 mContentParent 的，初始化完成之后，就讲我们要显示的页面加入 mConentParent 里面，这时这个用于显示一个页面的窗口的整个布局层次从上到下就算是完整了，就可以用于显示我们要显示的内容了。

### 流程总结

当需要为 Activity 设置一个页面布局的时候，使用 setContentView(int layoutResource) 为 Activity 添加这个布局，然后就会调用到 PhoneWindow 的 setContentView 方法，在这个方法里，会初始化 mDecor 和 mContentParent ，他们分别是 DecorView 和 FrameLayout ，mDecor 是最顶级 View ，它向外与 Window 相连，向内包含要展示的布局，mContentParent 是 mDecor 的子 View ，而所要加载的布局也会加载到 mContentParent 上，最终完成这个过程。