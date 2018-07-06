---
title: Android arch components 源码分析（2）—— Lifecycle
date: 2018-07-06 11:10:47
categories: Android
tags: Android arch
---

`Lifecycle` 的实现跟 `ViewModel` 类似，都是利用 `Fragment` 来实现它的功能。通过添加一个 `fragment` 到 `activity` 中，这个 `fragment` 便能够接收到各个生命周期回调。

> 以下源码使用 1.1.1 版本


## 使用方法简介

这里我并不打算将太多 lifecycle 的用法，不熟悉的同学，可以参考[这里](https://developer.android.google.cn/topic/libraries/architecture/)。

为了使用 lifecycle，首先需要获取到一个 `LifecycleOwner`。
```Java
lifecycleOwner.getLifecycle().addObserver(observer);


public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

使用 support 包时，`AppCompatActivity` 就是一个 `LifecycleOwner`。具体的实现是 `SupportActivity`：
```Java
public class SupportActivity extends Activity implements LifecycleOwner {}
```

下面，我们就从 `SupportActivity` 开始分析 lifecycle 组件的实现。


## 获取生命周期

```Java
public class SupportActivity extends Activity implements LifecycleOwner {

    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 初始化 ReportFragment
        ReportFragment.injectIfNeededIn(this);
    }
}
```

可以看到，在上一节中我们执行的 `lifecycleOwner.getLifecycle()` 返回的，就是 `mLifecycleRegistry`。关于 `LifecycleRegistry`，我们在下一节再看，这里先看 `ReportFragment`。

`ReportFragment` 就是我们在一开始说的，用于获取生命周期的 `fragment`：
```Java
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "android.arch.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

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

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    private ActivityInitializationListener mProcessListener;

    private void dispatchCreate(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onCreate();
        }
    }

    private void dispatchStart(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onStart();
        }
    }

    private void dispatchResume(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onResume();
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        // 对于 SupportActivity 来说，执行的是下面这个
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }

    interface ActivityInitializationListener {
        void onCreate();

        void onStart();

        void onResume();
    }
}
```

`ReportFragment` 的实现很简单，读者自己看看就好。下面我们开始看不那么好理解的 `LifecycleRegistry`。


## 生命周期事件的分发

在看代码前，我们先来了解一下 `Lifecycle` 的状态和事件：
```Java
public abstract class Lifecycle {

    public enum Event {
        /**
         * Constant for onCreate event of the {@link LifecycleOwner}.
         */
        ON_CREATE,
        /**
         * Constant for onStart event of the {@link LifecycleOwner}.
         */
        ON_START,
        /**
         * Constant for onResume event of the {@link LifecycleOwner}.
         */
        ON_RESUME,
        /**
         * Constant for onPause event of the {@link LifecycleOwner}.
         */
        ON_PAUSE,
        /**
         * Constant for onStop event of the {@link LifecycleOwner}.
         */
        ON_STOP,
        /**
         * Constant for onDestroy event of the {@link LifecycleOwner}.
         */
        ON_DESTROY,
        /**
         * An {@link Event Event} constant that can be used to match all events.
         */
        ON_ANY
    }

    /**
     * Lifecycle states. You can consider the states as the nodes in a graph and
     * {@link Event}s as the edges between these nodes.
     */
    public enum State {
        /**
         * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
         * any more events. For instance, for an {@link android.app.Activity}, this state is reached
         * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() onDestroy} call.
         */
        DESTROYED,

        /**
         * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
         * the state when it is constructed but has not received
         * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
         */
        INITIALIZED,

        /**
         * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} call;
         *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
         * </ul>
         */
        CREATED,

        /**
         * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onStart() onStart} call;
         *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
         * </ul>
         */
        STARTED,

        /**
         * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached after {@link android.app.Activity#onResume() onResume} is called.
         */
        RESUMED;

        /**
         * Compares if this State is greater or equal to the given {@code state}.
         *
         * @param state State to compare with
         * @return true if this State is greater or equal to the given {@code state}
         */
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```
`Lifecycle.Event` 对应 activity 的各个声明周期，`State` 则是 `Lifecycle` 的状态。在 `LifecycleRegistry` 中定义了状态间的转化关系：
```Java
public class LifecycleRegistry extends Lifecycle {

    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }

    private static Event downEvent(State state) {
        switch (state) {
            case INITIALIZED:
                throw new IllegalArgumentException();
            case CREATED:
                return ON_DESTROY;
            case STARTED:
                return ON_STOP;
            case RESUMED:
                return ON_PAUSE;
            case DESTROYED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private static Event upEvent(State state) {
        switch (state) {
            case INITIALIZED:
            case DESTROYED:
                return ON_CREATE;
            case CREATED:
                return ON_START;
            case STARTED:
                return ON_RESUME;
            case RESUMED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }
}
```
这三个方法，可以总结为下面这样一张图：
![android-arch-lifecycle-states](android-arch-lifecycle-states.png)

`downEvent` 在图中表示从一个状态到他下面的那个状态，`upEvent` 则是往上。


了解了 `Lifecycle` 的状态后，我们继续来看 `LifecycleRegistry`。上一节我们知道，activity 的生命周期发生变化后，会调用到 `LifecycleRegistry` 的 `handleLifecycleEvent`：
```Java
public class LifecycleRegistry extends Lifecycle {

    private int mAddingObserverCounter = 0;

    private boolean mHandlingEvent = false;
    private boolean mNewEventOccurred = false;

    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        // 当我们在 LifecycleRegistry 回调 LifecycleObserver 的时候触发状态变化时，
        // mHandlingEvent 为 true；
        // 添加 observer 的时候，也可能会执行回调方法，这时候如果触发了状态变化，
        // 则 mAddingObserverCounter != 0
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // 不需要执行 sync。
            // 执行到这里的情况是：sync() -> LifecycleObserver -> moveToState()
            // 这里直接返回后，还是会回到 sync()，然后继续同步状态给 observer
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        // sync() 会把状态的变化转化为生命周期事件，然后转发给 LifecycleObserver
        sync();
        mHandlingEvent = false;
    }
}
```

`LifecycleRegistry` 本来要做的事其实是很简单的，但由于他需要执行客户的代码，由此引入了很多额外的复杂度。原因是，客户代码并不处于我们的控制执行，他们可能做出任何可以做到的事。例如这里，在回调中又触发状态变化。类似的情况是，在持有锁的时候不调用客户代码，这个也会让实现变得比较复杂。

接下来我们看 `sync()`：
```Java
public class LifecycleRegistry extends Lifecycle {

    /**
     * Custom list that keeps observers and can handle removals / additions during traversal.
     *
     * 这个 Invariant 非常重要，他会影响到 sync() 的逻辑
     * Invariant: at any moment of time for observer1 & observer2:
     * if addition_order(observer1) < addition_order(observer2), then
     * state(observer1) >= state(observer2),
     */
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();


    // happens only on the top of stack (never in reentrance),
    // so it doesn't have to take in account parents
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            // mNewEventOccurred 是为了在 observer 触发状态变化时让 backwardPass/forwardPass()
            // 提前返回用的。我们刚准备调他们，这里设置为 false 即可。
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                // mObserverMap 里的元素的状态是非递增排列的，也就是说，队头的 state 最大
                // 如果 mState 小于队列里最大的那个，说明有元素需要更新状态
                // 为了维持 mObserverMap 的 Invariant，这里我们需要从队尾往前更新元素的状态
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            // 如果 mNewEventOccurred，说明在上面调用 backwardPass() 时，客户触发了状态修改
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }

    // 如果所有的 observer 的状态都已经同步完，则返回 true
    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true;
        }
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        // 因为我们保证队头的 state >= 后面的元素的 state，所以只要判断头尾就够了
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }

}
```

`sync()` 的主要作用就是根据把 `mObserverMap` 里所有元素的状态都同步为 `mState`。我们继续看剩下的 `backwardPass/forwardPass`：
```Java
public class LifecycleRegistry extends Lifecycle {

    private ArrayList<State> mParentStates = new ArrayList<>();


    private void forwardPass(LifecycleOwner lifecycleOwner) {
        // 从队头开始迭代
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    // 可能在回调客户代码的时候，客户把自己移除了
                    && mObserverMap.contains(entry.getKey()))) {
                // pushParentState 和 popParentState 我们下一小节再看，这里先忽略
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        // 从队尾开始迭代
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }

    private void popParentState() {
        mParentStates.remove(mParentStates.size() - 1);
    }

    private void pushParentState(State state) {
        mParentStates.add(state);
    }
}
```
在看这两个方法时，可以参考上面的状态图。比方说，假设当前队列里的元素都处于 `CREATED`。接着收到了一个 `ON_START` 事件，从图里面可以看出，接下来应该是要到 `STARTED` 状态。由于 `STARTED` 大于 `CREATED`，所以会执行 `forwardPass()`。`forwardPass()` 里面调用 `upEvent(observer.mState)`，返回从 `CREATED` 往上到 `STARTED` 需要发送的事件，也就是 `ON_START`，于是 `ON_START` 事件发送给了客户。


## 注册/注销 observer

注册 observer 由 `addObserver` 方法完成：
```Java
public class LifecycleRegistry extends Lifecycle {

    // 这段注释应该是这整个类里面最难理解的了吧，至少对于我来说是这样
    // we have to keep it for cases:
    // void onStart() {
    //     // removeObserver(this)，说明 onStart() 这个方法所属的类是一个 LifecycleObserver
    //     // 所以这里说的是，我们在回调里执行了下面两个操作
    //     mRegistry.removeObserver(this);
    //     mRegistry.add(newObserver);
    // }
    // 假定现在我们要从 CREATED 转到 STARTED 状态（也就是说，mState 现在是 STARTED）。
    // 这种情况下，只有将新的 observer 设置为 CREATED 状态，它的 onStart 才会被调用
    // 为了得到这个 CREATED，在这里才引入了 mParentStates。在 forwardPass 中执行
    // pushParentState(observer.mState) 时，observer.mState 就是我们需要的 CREATED。
    // backwardPass 的情况类似。
    // newObserver should be brought only to CREATED state during the execution of
    // this onStart method. our invariant with mObserverMap doesn't help, because parent observer
    // is no longer in the map.
    private ArrayList<State> mParentStates = new ArrayList<>();


    private State calculateTargetState(LifecycleObserver observer) {
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

        State siblingState = previous != null ? previous.getValue().mState : null;
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        // 返回最小的 state
        return min(min(mState, siblingState), parentState);
    }

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }

        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        // 在回调中执行了 addObserver()
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // 我们 dispatch 了一个事件给客户，在回调客户代码的时候，客户可能会修改我们的状态
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }


    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            // 由于篇幅有限，这里的 callback 就不看了。
            // 简单提一下，在使用 annotion 的时候，对应的 observer 会生成
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}
```

由于篇幅有限，这里的 callback 就不看了。简单提一下，在使用 annotion 的时候，对应的 observer 会生成一个 adapter，这个 adapter 会把对应的 `Lifecycle.Event` 装换为方法调用：
```Java
static class BoundLocationListener implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    void addLocationListener() {}

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    void removeLocationListener() {}
}


// 生成的代码
public class BoundLocationManager_BoundLocationListener_LifecycleAdapter implements GeneratedAdapter {
  final BoundLocationManager.BoundLocationListener mReceiver;

  BoundLocationManager_BoundLocationListener_LifecycleAdapter(BoundLocationManager.BoundLocationListener receiver) {
    this.mReceiver = receiver;
  }

  @Override
  public void callMethods(LifecycleOwner owner, Lifecycle.Event event, boolean onAny,
      MethodCallsLogger logger) {
    boolean hasLogger = logger != null;
    if (onAny) {
      return;
    }
    if (event == Lifecycle.Event.ON_RESUME) {
      if (!hasLogger || logger.approveCall("addLocationListener", 1)) {
        mReceiver.addLocationListener();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_PAUSE) {
      if (!hasLogger || logger.approveCall("removeLocationListener", 1)) {
        mReceiver.removeLocationListener();
      }
      return;
    }
  }
}
```

注销 observer 的实现就比较简单了：
```Java
public class LifecycleRegistry extends Lifecycle {
    @Override
    public void removeObserver(@NonNull LifecycleObserver observer) {
        // we consciously decided not to send destruction events here in opposition to addObserver.
        // Our reasons for that:
        // 1. These events haven't yet happened at all. In contrast to events in addObservers, that
        // actually occurred but earlier.
        // 2. There are cases when removeObserver happens as a consequence of some kind of fatal
        // event. If removeObserver method sends destruction events, then a clean up routine becomes
        // more cumbersome. More specific example of that is: your LifecycleObserver listens for
        // a web connection, in the usual routine in OnStop method you report to a server that a
        // session has just ended and you close the connection. Now let's assume now that you
        // lost an internet and as a result you removed this observer. If you get destruction
        // events in removeObserver, you should have a special case in your onStop method that
        // checks if your web connection died and you shouldn't try to report anything to a server.
        mObserverMap.remove(observer);
    }
}
```

恭喜你，相信你现在对 lifecycle 的实现已经胸有成竹，可以愉快地装逼了。

<br>

