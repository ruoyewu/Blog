---
title: Android SharedPreferences
date: 2019-03-09 10:00
tags:
	- android
---

SharedPreferences 是 Android 中一种轻量级的持久化存储方案，其本质是使用 XML 文件存储一系列的键值对，为了提高其使用效率，采用了异步写、内存缓存等方法。

### 关于调用

SharedPreferences 是一个接口，其实现类是 SharedPreferencesImpl ，它有一个构造方法`SharedPreferencesImpl(File, int)`，使用 protected 修饰，所以我们在使用它的时候一般都不能直接实例化一个 SharedPreferences ，而是通过 Context 的`getSharedPreferences(String, int)`得到其实例。在 ContextImpl 类中有关于这个方法的具体实现，实现分为两步：第一步将 name 转化为对应的 File 文件，第二步通过 File 和 Mode 得到对应的 SharedPreferences 实例。并且在这两步中都使用了 ArrayMap 做缓存，分别缓存了 File 和已经实例化的 SharedPreferences 。

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
    // At least one application in the world actually passes in a null
    // name.  This happened to work because when we generated the file name
    // we would stringify it to "null.xml".  Nice.
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }
    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }
    return getSharedPreferences(file, mode);
}

@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            checkMode(mode);
            if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                if (isCredentialProtectedStorage()
                        && !getSystemService(UserManager.class)
                                .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                    throw new IllegalStateException("SharedPreferences in credential encrypted "
                            + "storage are not available until after user is unlocked");
                }
            }
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}
```

通过两步缓存，使得我们在使用一个 SP 的时候能够尽可能不浪费时间，或者也可以在开发中直接使用一个 SharedPreferences 单例。

另外，SP 作为一种数据持久化存储方式，给开发者提供了一种非常便利的数据存取方式，例如取一个整形的时候只需要`sp.getInt(String, int)`即可，存一个整形的时候直接调用`sp.edit().putInt(String, int).apply()`，相比于其他的一些如 SQLite 存储、文件存储等都方便了很多，得益于在 SP 内部通过一个 HashMap 缓存了从 XML 文件中读取的键值对，也完成了对应的将一个 HashMap 中的值存储到 XML 文件中的功能。

再看 SP 的构造方法：

```java
@UnsupportedAppUsage
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}
```

-   mFile，对应的存储在磁盘中的 XML 文件
-   mBackupFile，用于提交更改时的备份文件
-   mMode，当前 XML 的读写权限，私有、公共读、公共写等
-   mLoaded，是否已经将数据从文件加载到内存
-   mMap，内存中保存的数据，取数据都是从这个 Map 中直接取
-   `startLoadFromDisk()`，加载文件的方法

每次实例化一个 SP 的时候，都会直接调用`startLoadFromDisk()`将文件中的数据加载到 mMap 中，而后续的诸如`getInt(String, int)`的方法都是从 mMap 中读取。

### 取数据

以取一个整形为例：

```java
@Override
public int getInt(String key, int defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Integer v = (Integer)mMap.get(key);
        return v != null ? v : defValue;
    }
}

@GuardedBy("mLock")
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```

SP 的读取操作都是一个同步代码块，这倒不是为了使多线程的读同步，而是为了使多线程的写同步。每当实例化一个 SP 的时候，它首先要做的是从对应的 XML 文件中读取数据到成员变量 mMap 中，这里的同步的目的就是为了让所有的取数据的操作发生在读取文件后面，也就是说只有先将数据从文件中读取到 mMap 中，才能从 mMap 中根据 key 取 value 。

读文件需要时间，所以 SP 中为读取 XML 文件新开了一个线程。

```java
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

那么 SP 的读数据操作是否会阻塞主线程呢？其实要看 SP 是否会阻塞主线程，只需要看是否在主线程中调用了阻塞线程的方法就行了。那么在上面的`awaitLoadedLocked()`方法中就可以看到，如果我们使用 SP 取数据的时候，文件还没有加载完成，那么此时即便是主线程也会陷入阻塞，直到文件加载完成。所以对于 SP 的使用来说，有两个基本的原则：

1.  不使用 SP 存储数据量过大的数据
2.  尽量不要在实例化 SP 之后立刻就调用 SP 的取数据方法

SP 在加载 XML 文件的时候是一次性将文件中的所有数据都加载到 HashMap 中的，所以如果数据量太大，一方面会导致对内存的占用过大，另一方面也需要太多的时间解析 XML 文件。另外，SP 会在实例化的时候调用`startLoadFromDisk()`方法加载 XML 文件，在加载文件的过程中会阻塞调用诸如`getInt(String)`的方法，就有可能导致主线程的阻塞，所以应该尽可能的早一些实例化 SP ，只要能够保证在取数据之前完成 XML 文件的加载，就不会造成阻塞。

### 存数据

SP 中存数据并不是直接由它自身完成，而是通过它的内部类 Editor 完成的，调用方法`edit()`得到一个 Editor 的实例：

```java
@Override
public Editor edit() {
    // TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    //      context.getSharedPreferences(..).edit().putString(..).apply()
    //
    // ... all without blocking.
    synchronized (mLock) {
        awaitLoadedLocked();
    }
    return new EditorImpl();
}
```

由于每次调用`edit()`都会直接实例化一个 Editor 类，所以在使用的过程中应该尽量少地调用`edit()`方法，如使用某种方法将 Editor 实例保存下来。Editor 的实现类是 EditorImpl ，它有两个成员变量：mModified 和 mClear 。

-   mModified，用于保存更改信息，更改信息不会直接写回 mMap 或文件，只有调用了`commit()`或`apply()`方法之后，更改才会生效
-   mClear，用于标识是否清除 mMap 的内容

以保存一个整形值为例：

```java
@Override
public Editor putInt(String key, int value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}
```

所有的存放的数据都保存在了 mModified 上面，而取数据都是从 mMap 中取的，这就是为什么不调用 Editor 的`commit()`或`apply()`方法，所有的修改都不会生效的原因。调用了`commit()`方法之后，才能将所做的修改保存到 XML 文件和 mMap 中去。

```java
@Override
public boolean commit() {
    long startTime = 0;
    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }
    MemoryCommitResult mcr = commitToMemory();
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

commit 方法主要完成了两个任务，一个是将修改提交到内存中，也就是提交给 mMap ，第二个任务就是将修改写回到文件中。对应的方法分别是`commitToMemory()`和`enqueueDiskWrite(MemoryCommitResult, Runnable)`。

#### `commitToMemory`，写回内存

```java
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;
    synchronized (SharedPreferencesImpl.this.mLock) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;
        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }
        synchronized (mEditorLock) {
            boolean changesMade = false;
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }
                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }
            mModified.clear();
            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }
            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```

`commitToMemory()`完成了将 mModified 和 mMap 合并的任务，还有对`OnSharedPreferencesChangeListener`的回调。还有一点，关于 clear 的判断，Editor 有一个`clear()`方法：

```java
@Override
public Editor clear() {
    synchronized (mEditorLock) {
        mClear = true;
        return this;
    }
}
```

它只有一个任务，将 mClear 置为 true ，而具体的相关的操作是在`commitToMemory()`方法中实现的，上述 26 行代码，如果 clear 为 true 就会将 mMap 清空，这就意味着当调用 Editor 的`clear()`方法的时候，它完成的任务仅仅是将 mMap 中的数据清空，而并不会对 mModified 中的数据造成影响。所以类似于`editor.putInt("ARG1", 1).clear();`的语句，并不能删除键值对`<ARG1, 1>`，所以在使用的时候应该明确自己要删除的内容再考虑应该使用`clear()`。

与之相对应的还有一个方法`remove(String)`：

```java
@Override
public Editor remove(String key) {
    synchronized (mEditorLock) {
        mModified.put(key, this);
        return this;
    }
}
```

这个方法的目的也是为了移除 mMap 中的数据，因为它并没有将这个 key 从 mModified 中移除，而是使用 value=this 对其进行标识，以便在`commitToMemory()`方法中识别。

`commitToMemory()`方法返回了一个 MemoryCommitResult 对象，这个对象保存了一些在写文件时需要用到的变量。

#### `enqueueDiskWrite(MemoryCommitResult, Runnable)`，写回文件

`enqueueDiskWrite(MemeryCommitResult, Runnable)`负责的是将修改写回到 XML 文件中：

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);
    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };
    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

这个方法提供了两种写文件的方式：直接在当前线程写文件；将写文件的任务放到 QueueWork 中执行。这对应着两种提交方法`commit()`和`apply()`。

首先从`commit()`方法说起，在这个方法中，一旦调用了`enqueueDiskWrite()`方法后，便调用了`mcr.writtenToDiskLatch.await()`方法，这个方法会阻塞当前线程。而在`commit()`中调用的`enqueueDiskWrite()`中有两种写方式：同步写和异步写。在某些情况下我们调用`commit()`也可能会执行异步写操作，但是在`commit()`方法中会阻塞当前线程直到写操作完成。

目前来说官方推荐使用`apply()`方法完成 Editor 的提交，原因就是`apply()`中没有了对上述操作的强行等待：

```java
@Override
public void apply() {
    final long startTime = System.currentTimeMillis();
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };
    QueuedWork.addFinisher(awaitCommit);
    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```

上面可以看到，在`apply()`方法中将`commit()`中用于等待写操作完成的代码放到了一个 Runnable 中执行，并在`enqueueDiskWrite()`方法中将这个 Runnable 连同写操作共同组成了一个新的 Runnable 放到了 QueueWork 中执行，此时写操作以及等待写操作的过程都在 QueueWork 对应的子线程中执行，在`apply()`方法中并没有阻塞线程的操作，这就是`apply()`方法优于`commit()`方法的原因。

另外为了避免可能出现某些问题，`apply()`方法还将这个等待 Runnable 加入了 QueueWork 的 Finisher 队列，一般用于即将结束某个 Activity 或 Service 的时候，要等待完成这里的写文件操作之后才行。

总结来说，就是`apply()`和`commit()`方法都是为了执行写文件操作而做的一些准备操作，它们分别使用了异步和同步的方式会写，最终都会调用`writeToFile(MemoryCommitResult, boolean)`完成具体的写操作。

`writeToFile()`的执行步骤大致如下：

1.  判断是否需要回写
2.  将 XML 文件改名为备份文件，然后将数据内容写入 XML 文件
3.  如果写入成功，则删除备份文件，否则在之后的`loadFromDisk()`方法中会将备份文件重新改名为 XML 文件

判断是否需要回写利用的是三个 stateGeneration 变量，分别对应着磁盘、内存和 MemoryCommitResult ，只有当前的 state 大于上一次提交到磁盘的 state 并且内存的 state 与 MemoryCommitResult 相同时，才会进行写操作，这能避免诸如`sp.edit().apply()`这种没有任何更改时不必要的写操作。

`writeToFile()`方法有三种结果：

1.  执行写操作，执行成功
2.  未执行写操作，执行成功（提交成功了但无更改）
3.  未执行写操作，执行失败

这些结果是通过`MemoryCommitResult.setDiskWriteResult(boolean, boolean)`返回的：

```java
void setDiskWriteResult(boolean wasWritten, boolean result) {
        this.wasWritten = wasWritten;
    writeToDiskResult = result;
    writtenToDiskLatch.countDown();
}
```

可以看到，无论执行成功还是失败，都会调用`writtenToDiskLatch.countDown()`方法，与之相对应的就是在`commit()`或`apply()`方法中的`mcr.writtenToDiskLatch.await()`方法，这一对方法的使用就是为了阻塞当前线程等待写操作完成。

至此，SP 的写数据操作也基本完成了，大致就是两个步骤：

1.  将修改写回内存
2.  将修改写回文件

在写文件方面，分别提供了异步写和同步写两种方法。

### 总结

SP 是 Android 中一种轻量级的持久化存储方案，为了能够更加有效的使用它，应该有以下几点需要注意：

1.  不要使用 SP 存储太大/太多的数据
2.  实例化 SP 之后隔一段时间再取数据
3.  尽量少调用`edit()`
4.  多使用`apply()`
5.  正确使用`clear()`方法