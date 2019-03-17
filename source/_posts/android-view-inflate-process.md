---
title: Android View 加载流程
date: 2019-03-14 12:30
tags:
	- android
---

一般情况下， Activity 会绑定一个 xml 文件，作为 Activity 展示的页面，对应的方法为`setContentView(int)`或者是`setContentView(View)`，以前者为例，这个方法是如何将一个 xml 文件展示出来的？下面就追本溯源，寻其根本。

### `Activity.setContentView(int)`

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

参考[Android 视图层次](../android-view-layout-level)，得知这里调用的是 PhoneWindow 的`setContentView(int)`方法。

### `Window.setContentView(int)`

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

两步操作：

1.  使 mContentParent 就绪
2.  利用 LayoutInflater 将 xml 布局文件加载到 mContentParent 中

对于上述第一步，如果 mContentParent 不存在，需要调用`installDecor()`方法，内部两个任务：

1.  判断 mDecor 是否存在，不存在则调用`generateDecor(int)`生成
2.  判断 mContentParent 是否存在，不存在则调用`generateLayout(DecorView)`生成

对于上述第二步，主要涉及到 LayoutInflater 这个类，它本质上的作用就是将一个 xml 文件实例化成为对应的 View 对象，所以这个类普遍应用于界面的加载中。一般使用它的步骤为`LayoutInflater.from(Context).inflate(int, ViewGroup, boolean)`，注意，这里使用`from(Context)`获取实例，而不是直接调用构造函数，由下面可以看出，我们常用的 LayoutInflater 是一个系统服务。

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

### `LayoutInflater.inflate(int, ViewGroup, boolean)`

解析 xml 文件的操作最终会调用这个方法，参数有三个：

1.  resource: int，对应的 xml 文件的资源索引
2.  root: ViewGroup，加载的这个 xml 布局的父 view
3.  attachToRoot: boolean，将 xml 布局加载出来之后是否直接添加到父 view 上

为什么要第二和第三个参数需要同时存在？第三个参数为 false 而第二个参数不为 null 时有什么意义？从个人使用上来说有一原因，在加载出来的 xml 的绘制流程 measure 过程中，如果设置了最外层 view 的宽度为 match_parent 时，需要依赖于父 view 的宽度，此时这里的 root 就有了作用。

这个方法有两个步骤：

1.  检测是否已有预编译，如果有直接使用，`tryInflatePrecompiled(int, Resources, ViewGroup, boolean)`
2.  对 xml 文件进行解析得到 view，`inflate(XmlPullParser, ViewGroup, boolean)`

预编译中使用到了反射，具体的不清楚。

然后就是使用 XmlResourcePraser 解析 xml 文件，并将解析出来的信息还原成对应的 View 。

### `LayoutInflater.inflate(XmlPullParser, ViewGroup, boolean)`

先列代码：

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;
        try {
            advanceToRootNode(parser);
            final String name = parser.getName();
            
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;
                if (root != null) {
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);
                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        return result;
    }
}
```

这个方法是加载 xml 布局的起点，它有两个分支：

1.  如果当前的标签是`<merge/>`，则直接调用`rInflate()`方法
2.  否则，调用`createViewFromTag()`方法实例化当前标签对应的 View ，再调用`rInflateChindren()`方法

因为`merge`标签不是一个特定的 View ，而是一种布局上的优化，没有对应的 View ，直接加载子 View 即可。

### `LayoutInflater.createViewFromTag()`

由方法名就可以知道，这个方法完成了从 tag 到 View 的转变，首先看代码：

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }
    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }
    try {
        View view = tryCreateView(parent, name, context, attrs);
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
        return view;
    } catch (InflateException e) {
        throw e;
    } catch (ClassNotFoundException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } catch (Exception e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + name, e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    }
}
```

基本流程如下：

1.  是否加载相关的 theme
2.  调用`tryCreateView()`创建 View，如果失败，转 3
3.  调用`onCreateView()`和`createView()`创建 View

第一个关于主题的没什么好说，在第二、三步里出现了三个加载 View 的方法，它们都可以用来加载一个 View ，那么它们之间的区别是什么？这个应该直接从源码中寻找。首先看`tryCreateView()`的功能。

#### `tryCreateView()`

代码为：

```java
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }
    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }
    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }
    return view;
}
```

也有几种分支，不过都是调用了 LayoutInflater.Factory 这个工厂接口的`onCreateView()`方法。关于这个方法的注释为：

>   Hook you can supply that is called when inflating from a LayoutInflater. You can use this to customize the tag names available in your XML layout files.

大意就是可以自定义 xml 文件中 tag 的名字，然后实现这个方法，根据自定义的名字创建对应的 View 。举个例子，如 LayoutInflater.Factory 的一个实现类 AppCompatDelegateImpl 的`onCreateView()`方法：

```java
public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return createView(parent, name, context, attrs);
}
public View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs) {
    // 省略若干
    return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
            IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
            true, /* Read read app:theme as a fallback at all times for legacy reasons */
            VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
    );
}

// AppCompatViewInflater.createView() 方法代码
final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    final Context originalContext = context;
    // We can emulate Lollipop's android:theme attribute propagating down the view hierarchy
    // by using the parent's context
    if (inheritContext && parent != null) {
        context = parent.getContext();
    }
    if (readAndroidTheme || readAppTheme) {
        // We then apply the theme on the context, if specified
        context = themifyContext(context, attrs, readAndroidTheme, readAppTheme);
    }
    if (wrapContext) {
        context = TintContextWrapper.wrap(context);
    }
    View view = null;
    // We need to 'inject' our tint aware Views in place of the standard framework versions
    switch (name) {
        case "TextView":
            view = createTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageView":
            view = createImageView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Button":
            view = createButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "EditText":
            view = createEditText(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Spinner":
            view = createSpinner(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageButton":
            view = createImageButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckBox":
            view = createCheckBox(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RadioButton":
            view = createRadioButton(context, attrs);
            verifyNotNull(view, name);
            break;
        case "CheckedTextView":
            view = createCheckedTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "AutoCompleteTextView":
            view = createAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "MultiAutoCompleteTextView":
            view = createMultiAutoCompleteTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "RatingBar":
            view = createRatingBar(context, attrs);
            verifyNotNull(view, name);
            break;
        case "SeekBar":
            view = createSeekBar(context, attrs);
            verifyNotNull(view, name);
            break;
        default:
            // The fallback that allows extending class to take over view inflation
            // for other tags. Note that we don't check that the result is not-null.
            // That allows the custom inflater path to fall back on the default one
            // later in this method.
            view = createView(context, name, attrs);
    }
    if (view == null && originalContext != context) {
        // If the original context does not equal our themed context, then we need to manually
        // inflate it using the name so that android:theme takes effect.
        view = createViewFromTag(context, name, attrs);
    }
    if (view != null) {
        // If we have created a view, check its android:onClick
        checkOnClickListener(view, attrs);
    }
    return view;
}
```

从上面列出的方法就可见一斑，根据解析出了 xml 的 tag 的名字，直接创建出不同的实例，如果开发者有一些自己的需求，就可以单独实现 LayoutInflater.Factory 接口，并通过`LayoutInflater.setPrivateFactory()`方法将自定义的工厂类加进去。

为了避免开发者随意添加工厂类破坏系统本身的加载（例如自定义一个加载器，将 tag 名字为`TextView`加载出一个 ImageView 等），使用了系统加载和私人加载两种加载模式，对于任何一个加载工作，会优先使用系统加载器，如此便可以阻止开发者的恶意破坏。类似于 Java  中类加载机制。

为了供开发者使用多个加载器，还增加了 FactoryMerger 类，它内部保存了多个 Factory 实现类，其本身也是 Factory 的实现类。如下：

```java
private static class FactoryMerger implements Factory2 {
    private final Factory mF1, mF2;
    private final Factory2 mF12, mF22;
    FactoryMerger(Factory f1, Factory2 f12, Factory f2, Factory2 f22) {
        mF1 = f1;
        mF2 = f2;
        mF12 = f12;
        mF22 = f22;
    }
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        View v = mF1.onCreateView(name, context, attrs);
        if (v != null) return v;
        return mF2.onCreateView(name, context, attrs);
    }
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        View v = mF12 != null ? mF12.onCreateView(parent, name, context, attrs)
                : mF1.onCreateView(name, context, attrs);
        if (v != null) return v;
        return mF22 != null ? mF22.onCreateView(parent, name, context, attrs)
                : mF2.onCreateView(name, context, attrs);
    }
}
```

#### `onCreateView()`

先看代码：

```java
// in createViewFromTag()
if (-1 == name.indexOf('.')) {
    view = onCreateView(parent, name, attrs);
} else {
    view = createView(name, null, attrs);
}

protected View onCreateView(View parent, String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(name, attrs);
}

protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return createView(name, "android.view.", attrs);
}
```

在编写 xml 布局文件的时候，有一条规则：

1.  如果一个 View 在 android.view 包下，就可以不需要添加包名，直接使用 View 的名字
2.  否则就需要引用这个 View 完整的路径名

其原因就在这里，代码中自动对不完全的路径进行了补全，其原因应该是考虑到 android.view 包下的 View 们使用的太过频繁，每一个 View 都增加完整的路径会造成不必要的重复操作，故而有了这样一个机制。除了那些可以使用 Factory 加载出来的 View 之外，其他的 View 都需要其完整的路径，而路径的目的，就是为了反射。

#### `createView()`

方法代码如下：

```java
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // If we have a filter, apply it to cached constructor
            if (mFilter != null) {
                // Have we seen this name before?
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // New class -- remember whether it is allowed
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);
                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }
        Object lastContext = mConstructorArgs[0];
        if (mConstructorArgs[0] == null) {
            // Fill in the context if not already within inflation.
            mConstructorArgs[0] = mContext;
        }
        Object[] args = mConstructorArgs;
        args[1] = attrs;
        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        mConstructorArgs[0] = lastContext;
        return view;
    } catch (NoSuchMethodException e) {
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Error inflating class " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } catch (ClassCastException e) {
        // If loaded class is not a View subclass
        final InflateException ie = new InflateException(attrs.getPositionDescription()
                + ": Class is not a View " + (prefix != null ? (prefix + name) : name), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } catch (ClassNotFoundException e) {
        // If loadClass fails, we should propagate the exception.
        throw e;
    } catch (Exception e) {
        final InflateException ie = new InflateException(
                attrs.getPositionDescription() + ": Error inflating class "
                        + (clazz == null ? "<unknown>" : clazz.getName()), e);
        ie.setStackTrace(EMPTY_STACK_TRACE);
        throw ie;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

一个标准的反射过程，首先取得对应 View 类的 Contructor ，这里有使用 HashMap 作了一级缓存。然后调用`Constructor.newInstance()`方法构建出 View 实例。

至此，`createViewFromTag()`方法就执行完了，再回到`inflate()`方法中去，会看到，又继续调用了`rInflateChildren()`方法。

### `LayoutInflater.rInflate()`

代码如下：

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}

void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
        if (type != XmlPullParser.START_TAG) {
            continue;
        }
        final String name = parser.getName();
        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            throw new InflateException("<merge /> must be the root element");
        } else {
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }
    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }
    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

这是一个递归方法，对应着 xml 的递归结构，它有五个参数：

1.  parser: XmlPullParser，xml 解析器，用于读取解析内容
2.  parent: View，当前层所有 view 的父 view
3.  context: Context，用于请求资源
4.  attrs: AttributeSet，用于读取 xml 中 view 的配置信息
5.  finishInflate: boolean，是否结束当前 view 的解析

内部有一个 while 循环，用于解析出当前层所有的 view ，也就是 parent 的所有子 view ，每个循环中先通过`createViewFromTag()`方法求出一个子 view ，然后接着调用`rInflateChindren()`方法求这个子 view 的子 view ，构成递归，并将最后求出的这个子 view 加到 parent 里面，执行结束。

执行完这一步之后，`inflate()`方法也就执行结束了，由 xml 文件表示的布局转变成了对应的 View 结构。但是光有 view 不行，只有将 view 代表的内容画出来，才能被用户看到。这一过程，具体看[Android View 绘制流程](../android-view-draw-process)。