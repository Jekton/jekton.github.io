---
title: Android arch components 源码分析（3）—— LiveData
date: 2018-07-14 15:42:06
categories: Android
tags: Android arch
---

本篇我们来看看 Android 架构组件中的 `LiveData` 。跟 `ViewModel` 相比，`LiveData` 具有生命周期感知能力，也就是说，他把 `ViewModel` 和 lifecycle 结合了起来。当应用的数据有更新时，一般我们仅希望应用对用户可见时才更新 UI；更进一步，如果应用不可见，我们甚至可以停止数据的更新。这就是所谓的“感知应用的生命周期”。

这里我们主要关注 `LiveData` 的实现，用法可以参考 Google 的[教程](https://developer.android.google.cn/topic/libraries/architecture/)。


## 添加 Observer

使用 `LiveData` 时，首先要做的，就是添加一个 `Observer<T>`。
```Java
public interface Observer<T> {
    /**
     * Called when the data is changed.
     * @param t  The new data
     */
    void onChanged(@Nullable T t);
}

// 注意，他是 abstract class
public abstract class LiveData<T> {

    // 只有 onStart 后，对数据的修改才会触发 observer.onChanged()
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {}

    // 无论何时，只要数据发生改变，就会触发 observer.onChanged()
    public void observeForever(@NonNull Observer<T> observer) {}
}
```

由于 `LiveData` 是一个 `abstract class`，我们不能直接生成他的实例。对于数据的*拥有者*，可以使用 `MutableLiveData`：
```Java
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        // LiveData.postValue() 是一个 protected 方法
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        // LiveData.setValue() 是一个 protected 方法
        super.setValue(value);
    }
}
```

> 所谓数据的拥有者。举个例子，你使用的是 MVP 模式，那么数据就属于 Model 层，另外两层不应该修改数据。

通过让这两个 setter 方法成为 `protected`，只要我们给客户返回的是 `LiveData`，就不用担心数据会被客户意外修改：
```Java
class SomeClass extends ViewModel {

    // 在类的内部持有的是 MutableLiveData，所以我们可以调用 postValue/setValue
    private final MutableLiveData<Foo> mYourData = new MutableLiveData<>();

    // 返回的是 LiveData，LiveData 的 public 方法中并没有 postValue/setValue
    public LiveData<Foo> getData() {
        return mYourData;
    }
}
```

> 活用 `public`, `protected`, `private`, `default access` 和 `final` 可以让我们的设计意图更加清晰。


现在回到我们的 `observe()` 方法，`observeForever` 的实现跟 `observe` 是类似的，我们就不看了，这里只看 `observe()`：
```Java
public abstract class LiveData<T> {

    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        // activity 已经 destroy，也就没必要添加 observer 了
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            // 同一个 observer，只有对应的 lifecycleOwner 不一样，才可以重新添加
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        // 这种方式添加的 observer，只有 activity 可见时才会收到数据更新的通知，
        // 为了知道什么时候 activity 是可见的，这里需要注册到 Lifecycle。
        // 也是因为这个，observe() 比 observeForever() 多了一个参数 lifecycleOwner
        owner.getLifecycle().addObserver(wrapper);
    }

}
```

我们继续看 `LifecycleBoundObserver`：
```Java
public abstract class LiveData<T> {

    // 空实现，如果在 LiveData 变为 inactive 状态后想停止更新数据，可以
    // override 这两个方法
    protected void onActive() {}
    protected void onInactive() {}

    private abstract class ObserverWrapper {
        final Observer<T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<T> observer) {
            mObserver = observer;
        }

        // 如果 observer 处于 active 状态，则返回 true
        abstract boolean shouldBeActive();

        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }

        void detachObserver() {
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            // LiveData.this.mActiveCount 表示处于 active 状态的 observer 的数量
            // 当 mActiveCount 大于 0 时，`LiveData` 处于 active 状态
            // 注意区分 observer 的 active 状态和 LiveData 的 active 状态
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            if (wasInactive && mActive) {
                // inactive -> active
                onActive();
            }
            // 这里用 else if 比较好，因为只有一个会执行。else if 更易读
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                // mActiveCount 在我们修改前等于 1，也就是说，`LiveData` 从 active
                // 状态变到了 inactive
                onInactive();
            }
            if (mActive) {
                // observer 从 inactive 到 active，此时客户拿到的数据可能不是最新的，这里需要 dispatch 一下
                // 关于他的实现，我们下一节再看
                dispatchingValue(this);
            }
        }
    }

    class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
        @NonNull final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            // onStart 到 onStop 之间则认为是 active 状态
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        // 这个是 lifecycle 的回调函数
        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            // 刚刚生成 LifecycleBoundObserver 的实例时，mActive == false，注册到
            // Lifecycle 后，Lifecycle 会同步状态给我们（也就是回调本函数）。
            // 不熟悉 lifecycle 的读者，可以看
            // https://jekton.github.io/2018/07/06/android-arch-lifecycle/
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
}
```

到这里，observer 的注册我们就看完了。下面我们看看如何发布(publish)数据给 `LiveData`。


## 发布修改

要修改 `LiveData`，有两种方式：
```Java
public abstract class LiveData<T> {

    // 同步修改数据
    protected void setValue(T value);

    // 会用 Handler post 一个 runnable，然后在 runnable 里面 setValue
    protected void postValue(T value);
}
```

`setValue` 比较简单，我们先看 `setValue`：
```Java
public abstract class LiveData<T> {

    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        // 每次更新 value，都会使 mVersion + 1
        // ObserverWrapper 也有一个字段，叫 mLastVersion
        // 通过比较这两个字段，可以避免重复通知客户（具体在后面会看到）
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }

    // 如果 initiator == null，表示要通知所有的 observer
    // 不等于 null 则只通知 initiator
    private void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            // 在 observer 的回调里面又触发了数据的修改
            // 设置 mDispatchInvalidated 为 true 后，可以让下面的循环知道
            // 数据被修改了，从而开始一轮新的迭代。
            //
            // 比方说，dispatchingValue -> observer.onChanged -> setValue
            //            -> dispatchingValue
            // 这里 return 的是后面那个 dispatchingValue，然后在第一个
            // dispatchingValue 会重新遍历所有的 observer，并调用他们的
            // onChanged。
            //
            // 如果想避免这种情况，可以在回调里面使用 postValue 来更新数据
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                // 调用 observer.onChanged()
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        // 某个客户在回调里面更新了数据，break 后，这个 for 循环会
                        // 重新开始
                        break;
                    }
                }
            }
        // 当某个客户在回调里面更新了数据，mDispatchInvalidated == true
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
}
```

看过我那篇 lifecycle 源码分析的读者应该对 `dispatchingValue` 处理循环调用的方式很熟悉了。以这里为例，为了防止循环调用，我们在调用客户代码前先置位一个标志（`mDispatchingValue`），结束后再设为 `false`。如果在回调里面又触发了这个方法，可以通过 `mDispatchingValue` 来检测。

检测到循环调用后，再设置第二个标志（`mDispatchInvalidated`），然后返回。返回又会回到之前的调用，前一个调用通过检查 `mDispatchInvalidated`，知道数据被修改，于是开始一轮新的迭代。


下面是 `considerNotify`：
```Java
public abstract class LiveData<T> {

    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        // 对于 LifecycleBoundObserver 来说，即使 `LiveData` 的数据没有变化，只要 activity 的生命
        // 周期发生了改变，还是可能会调用 considerNotify 多次
        // 通过比较 observer.mLastVersion 和 mVersion，就能够知道 observer 是否已经拥有了最新的数据
        //
        // 实际上，observer.mLastVersion 最多只能等于 mVersion
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //noinspection unchecked
        observer.mObserver.onChanged((T) mData);
    }

}
```

看完了 `setValue`，`postValue` 对我们来说就很简单了：
```Java
public abstract class LiveData<T> {

    // 注意，他是 volatile。因为 postValue 可以从后台线程调用，
    private volatile Object mPendingData = NOT_SET;

    private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            //noinspection unchecked
            setValue((T) newValue);
        }
    };


    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            // 已经有一个 post 后还没有执行的 runnable，所以就不需要再 post 了，
            // 前面 post 的 runnable 执行时，会拿到这个新设置的 value
            return;
        }
        // 最终执行的就是 handler.post()
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }
}

```

`LiveData` 的核心代码我们已经看完了，其实他的实现也挺简单的，对吧？

## 总结

关于 `LiveData`，有两个值得我们学习的，一个是循环调用的处理，另一个是 `mVersion` 的使用。关于 `mVersion`，这里再举一个之前工作中遇到的例子。在后台线程对数据进行持久化的时候（这个线程拷贝了一份数据），数据还有可能会被更新。为了判断所保存的数据是不是最新的，我当时的做法就是引入一个类似 `mVersion` 的东西，每次修改数据，都把 `mVersion` 加 1。通过比较 `mVersion` 和所保存的数据的 version，就能够知道是不是保存了最新的数据（当然，更好的做法是告诉后台线程数据已经修改，让他重新拿一次数据）。
