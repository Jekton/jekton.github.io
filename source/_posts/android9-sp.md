---
title: Android P 源码分析 3 - SharedPreferences 源码分析
date: 2019-03-19 19:30:50
categories: Android
tags: [Android source]
---

本来按顺序这一篇应该是 logd，但突然有点好奇 SP 在保存数据的时候是怎么同步的，就还是先看 SP 吧，当做在开始啃 logd 这个硬骨头前轻松一下（虽然这么说，SP 还是有很多值得我们学习的地方的）。

<!-- more -->

## 获取 SP 实例

我们通过调用 `Context.getSharedPreferences` 获取一个 SharedPreferences 实例的时候，真正的实现在 `ContextImpl`：
```Java
// base/core/java/android/app/ContextImpl.java
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

/**
 * Map from package name, to preference name, to cached preferences.
 */
@GuardedBy("ContextImpl.class")
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

@GuardedBy("ContextImpl.class")
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}
```
这三个方法的实现都相当的直观，唯一有趣的是 `getSharedPreferencesCacheLocked` 里面那个 `packageName`。我们知道，一个应用的包名并不会改变；在访问内存中数据时，不同进程也不会互相干扰。这样看来，用 packageName 做 key 的这个 `sSharedPrefsCache` 是否有点多余？

通过查看 git 提交记录 `8e3ddab` 可以看到这样一句说明：

> Otherwise multiple applications using the same process can end up leaking SharedPreferences instances between the apps

其实 Android 有一个相当不常用的特性——多个应用可以共用同一个进程。在这种情况下，这里用 package name 就能够把各个应用的 SP 区分开。

这里的实现还隐含了 SP 的一个特性：一旦数据加载到内存，除非我们删除整个 SP，内存中的数据在整个进程的生命周期中都存在。正常情况下，SP 中的数据量是非常小的，这个并不会导致什么问题。

## SP 的初始化

还是跟前面一样，我们直接看代码：
```Java
// base/core/java/android/app/SharedPreferencesImpl.java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}

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

可以看到，SP 一创建就开始在后台加载数据了。利用这个特性，对于比较大的 SP 并且预期很快就要用到，可以提前获取 SP 实例，以触发他的初始化。这样一来，在随后我们真正需要读取里面的数据时，他很可能就已经加载完成，从而避免了第一次读取时的卡顿。

```Java
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        // It's important that we always signal waiters, even if we'll make
        // them fail with an exception. The try-finally is pretty wide, but
        // better safe than sorry.
        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    // 文件的最后修改时间
                    mStatTimestamp = stat.st_mtim;
                    // 文件大小
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll();
        }
    }
}
```

`Os.stat` 用来获取文件的元信息，它不是 JDK 提供的 API。关于它的实现，有兴趣的读者可以参考《UNIX 环境高级编程》（APUE）。

XML 的解析并不是我们关心的东西，只要知道 SP 是用 XML 文件存储的就好。

加载成功后的 `notifyAll` 我们要结合 `awaitLoadedLocked` 来看。在我们准备读、写 SP 的时候，都会先调用 `awaitLoadedLocked` 等待 `loadFromDisk`。`loadFromDisk` 最后的 `notifyAll` 就是为了唤醒这些等待数据加载完成的线程。
```Java
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

`mThrowable` 是在数据加载失败时由 `loadFromDisk` 设置的。这里相当于把后台的数据加载线程发生的异常转移到了实际需要读写 SP 的线程，有一定的借鉴的意义。


## 从 SP 中读取数据

读数据的情况很简单，只需要等 `loadFromDisk` 加载完数据，然后直接从 map 里面 get 就可以了：
```Java
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

读其他类型的情况类似，这里就不看了。


## 向 SP 写入数据

写数据的时候，我们要先获取一个 `Editor`:
```Java
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

public final class EditorImpl implements Editor {
    private final Object mEditorLock = new Object();

    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();

    @GuardedBy("mEditorLock")
    private boolean mClear = false;

    @Override
    public Editor putString(String key, @Nullable String value) {
        synchronized (mEditorLock) {
            mModified.put(key, value);
            return this;
        }
    }

    @Override
    public Editor remove(String key) {
        synchronized (mEditorLock) {
            mModified.put(key, this);
            return this;
        }
    }

    @Override
    public Editor clear() {
        synchronized (mEditorLock) {
            mClear = true;
            return this;
        }
    }
    
    // ......
}
```

`EditorImpl` 把所有的修改都保存在成员变量 `mModified` 和 `mClear` 里，以达到批量修改的目的。下面我们看看他的 `apply` 方法的实现：
```Java
public final class EditorImpl implements Editor {

    // ...

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
}
```

这里分是三个步骤：
1. 把修改写到内存的缓存里
2. 把修改写到硬盘（文件）
3. 通知监听者

下面我们一个一个步骤来看：

### 把修改写到内存的缓存里

这一步是由 `commitToMemory` 实现的：
```Java
public final class EditorImpl implements Editor {
    // ...

    // Returns true if any changes were made
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
}
```

`mDiskWritesInFlight` 是 `SharedPreferencesImpl` 的成员变量，表示当前有多少个正着执行中的硬盘写操作。如果我们不是唯一的写者，表示在前面有某个写操作正把 `mMap` 的内容写到硬盘。此时我们不能直接修改 `mMap`，否则硬盘的数据的一致性会有问题（比方说，部分 key 是旧的，部分是新的）。拷贝一份 `mMap` 后，我们就可以安全地进行修改了。

SP 提供了一个 `registerOnSharedPreferenceChangeListener` 方法，通过它我们可以注册监听器，在 SP 修改的时候得到通知。相关的 listener 就放在 `mListener` 里。

我们可以把这个方法里的 `mapToWriteToDisk` 看做是 SP 的一个快照（snapshot）， `SharedPreferencesImpl::mCurrentMemoryStateGeneration` 用来跟踪这些快照的年龄。当我们往文件里面写入数据的时候，只有年龄最大（数据最新）的那一个快照才需要写到硬盘里（旧的数据即使写了进入，也马上会被覆盖）。关于这一点，在后面我们看 `writeToFile` 实现的时候就知道了。

方法最后返回的 `MemoryCommitResult` 就很简单了，只是一些数据的聚集：
```Java
// Return value from EditorImpl#commitToMemory()
private static class MemoryCommitResult {
    final long memoryStateGeneration;
    @Nullable final List<String> keysModified;
    @Nullable final Set<OnSharedPreferenceChangeListener> listeners;
    final Map<String, Object> mapToWriteToDisk;
    final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

    @GuardedBy("mWritingToDiskLock")
    volatile boolean writeToDiskResult = false;
    boolean wasWritten = false;

    private MemoryCommitResult(long memoryStateGeneration, @Nullable List<String> keysModified,
            @Nullable Set<OnSharedPreferenceChangeListener> listeners,
            Map<String, Object> mapToWriteToDisk) {
        this.memoryStateGeneration = memoryStateGeneration;
        this.keysModified = keysModified;
        this.listeners = listeners;
        this.mapToWriteToDisk = mapToWriteToDisk;
    }

    void setDiskWriteResult(boolean wasWritten, boolean result) {
        this.wasWritten = wasWritten;
        writeToDiskResult = result;
        writtenToDiskLatch.countDown();
    }
}
```

### 把修改写到硬盘（文件）

为了帮助你回忆 `apply` 的工作，这里我再拷贝一份他的源码。
```Java
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

`QueuedWork` 是一个全局的工作队列，`addFinisher` 添加进去的 runnable 不会立即执行，仅仅是放到一个链表里。当某个人想要等待 `QueuedWork` 所有工作执行完毕时，就调用 `QueuedWork.waitToFinish()`，在这个方法里面会取出早先所有 `addFinisher` 的任务，一个一个执行。

在 `awaitCommit` 里面，我们调用了 `mcr.writtenToDiskLatch.await()` 来等待数据写入硬盘，所以这里的把 `awaitCommit` 放到 `QueuedWork` 里，就提供了一种机制，让外界等待文件的写入操作。读者可以到 `ActivityThread` 中搜一下 `QueuedWork.waitToFinish`，会发现在 activity/service stop 的时候，都会执行这个操作，从而保证在应用退出前 SP 已经写入硬盘。

接下来的 `enqueueDiskWrite` 执行真正的写入操作：
```Java
/**
 * Enqueue an already-committed-to-memory result to be written
 * to disk.
 *
 * They will be written to disk one-at-a-time in the order
 * that they're enqueued.
 *
 * @param postWriteRunnable if non-null, we're being called
 *   from apply() and this is the runnable to run after
 *   the write proceeds.  if null (from a regular commit()),
 *   then we're allowed to do this disk write on the main
 *   thread (which in addition to reducing allocations and
 *   creating a background thread, this has the advantage that
 *   we catch them in userdebug StrictMode reports to convert
 *   them where possible to apply() ...)
 */
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
`isFromSyncCommit` 是我们直接调用 `editor.commit` 的情况，这个时候如果我们是唯一的写者（writter），就直接调用 `writeToDiskRunnable.run()` 执行写入操作。其他情况下，都放到 `QueuedWork` 里面执行。

`QueuedWork` 在内部使用一个 `HandlerThread` 串行地执行所有的工作，`queue()` 后的任务都会被 post 到这个线程去执行：
```Java
public class QueuedWork {
    /**
     * Queue a work-runnable for processing asynchronously.
     *
     * @param work The new runnable to process
     * @param shouldDelay If the message should be delayed
     */
    public static void queue(Runnable work, boolean shouldDelay) {
        Handler handler = getHandler();

        synchronized (sLock) {
            sWork.add(work);

            if (shouldDelay && sCanDelay) {
                handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
            } else {
                handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
            }
        }
    }
}
```
对 `editor.apply()` 而言，这里的 `shouldDelay` 参数为 `true`，实际的写入操作会延迟 100 毫秒才执行。接下来看完 `writeToFile` 的实现以后，我们就会发现，这个小小的延迟在频繁 `editor.apply` 的时候实际上有一定优化作用的。

```Java
@GuardedBy("mWritingToDiskLock")
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    boolean fileExists = mFile.exists();

    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        boolean needsWrite = false;

        // Only need to write if the disk state is older than this commit
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
            if (isFromSyncCommit) {
                needsWrite = true;
            } else {
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }

        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }

        boolean backupFileExists = mBackupFile.exists();

        if (DEBUG) {
            backupExistsTime = System.currentTimeMillis();
        }

        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);

        if (DEBUG) {
            outputStreamCreateTime = System.currentTimeMillis();
        }

        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

        writeTime = System.currentTimeMillis();

        FileUtils.sync(str);

        fsyncTime = System.currentTimeMillis();

        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        if (DEBUG) {
            setPermTime = System.currentTimeMillis();
        }

        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }

        if (DEBUG) {
            fstatTime = System.currentTimeMillis();
        }

        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();

        if (DEBUG) {
            deleteTime = System.currentTimeMillis();
        }

        mDiskStateGeneration = mcr.memoryStateGeneration;

        mcr.setDiskWriteResult(true, true);

        if (DEBUG) {
            Log.d(TAG, "write: " + (existsTime - startTime) + "/"
                    + (backupExistsTime - startTime) + "/"
                    + (outputStreamCreateTime - startTime) + "/"
                    + (writeTime - startTime) + "/"
                    + (fsyncTime - startTime) + "/"
                    + (setPermTime - startTime) + "/"
                    + (fstatTime - startTime) + "/"
                    + (deleteTime - startTime));
        }

        long fsyncDuration = fsyncTime - writeTime;
        mSyncTimes.add((int) fsyncDuration);
        mNumSync++;

        if (DEBUG || mNumSync % 1024 == 0 || fsyncDuration > MAX_FSYNC_DURATION_MILLIS) {
            mSyncTimes.log(TAG, "Time required to fsync " + mFile + ": ");
        }

        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }

    // Clean up an unsuccessfully written file
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```
首先我们看看 `needsWrite` 中 `mCurrentMemoryStateGeneration == mcr.memoryStateGeneration` 的情况，这个代码当前的快照是最新的，所以需要写入硬盘。之所以单独拿这个出来说，是为了说明前面 `QueuedWork.queue()` 里面那个 delay 所起到的作用。由于我们用的是异步的 `editor.apply`，所以这个延迟是隐含在 API 里的，还在正确的语义范畴里；另一方面，考虑应用频繁 `apply` 的情况，如果前后的 apply 间隔小于 100 毫秒，那么这个条件判断只在最后的写任务会为 `true`，从而避免了过多的无用的写硬盘操作。

最后我们看看 `mBackupFile` 的作用。在开头的 `loadFromDisk` 有这么一小段代码：
```Java
private void loadFromDisk() {
    synchronized (mLock) {
        // ...

        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }

        // ...
    }
}
```
如果备份文件存在，我们就把 `mFile` 删除，然后读备份文件的数据。

为了考察这个备份文件的作用，我们先假设 SP 是刚刚创建的，此时备份文件不存在，`writeToDisk` 先把 `mFile` 保存一个备份，然后往 `mFile` 写数据。在写数据成功的情况下，我们再删除前面的那个备份，此时只有 `mFile` 存在。

另一种可能性是，我们在写 `mFile` 的时候失败了，此时 `mFile` 里面是一些垃圾数据，而备份文件 `mBackupFile` 是我们在这个失败的写操作之前保存的，虽然它的信息不是最新的，却是完整的数据。在这种情况下，`loadFromDisk` 会加载备份文件的数据。

最后一种情况是我们在准备写数据的时候备份文件存在，这种只在前一次写文件失败的时候才会发生。此时 `mFile` 无疑是错误的，所以我们直接删掉它。


### 通知监听者

虽然文章已经很长，处于完整性考虑，还是把 `notifyListeners` 的代码也一起放上来。这里的实现很简单，我就不多说了。

```Java
public final class EditorImpl implements Editor {
    private void notifyListeners(final MemoryCommitResult mcr) {
        if (mcr.listeners == null || mcr.keysModified == null ||
            mcr.keysModified.size() == 0) {
            return;
        }
        if (Looper.myLooper() == Looper.getMainLooper()) {
            for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
                final String key = mcr.keysModified.get(i);
                for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
                    if (listener != null) {
                        listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
                    }
                }
            }
        } else {
            // Run this function on the main thread.
            ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
        }
    }
}
```