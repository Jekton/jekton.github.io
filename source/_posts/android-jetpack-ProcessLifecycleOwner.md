---
title: Android 源码秘密（1）—— ProcessLifecycleOwner
date: 2020-01-21 08:22:18
categories: Android
tags: [Android arch, ProcessLifecycleOwner]
---

尝试过事无巨细一行一行代码分析源码，也试过以感性的方式总结源码；第一种方式总觉得容易把读者绕进去，第二种则有些人看后觉得好像什么也没说。这里我尝试使用第三种方法，回到我们阅读源码的初衷——学习如何写代码，只是摘抄出源码里有教益意义的片段，来展示它所使用的小技巧。

作为第一个尝试，本篇文章我们研究的对象是 Android arch component 里的 `ProcessLifecycleOwner`。

## 如何无侵入地初始化库程序

通常情况下，如果我们的库函数需要在使用前先初始化一次，会对外保留一个 `init` 接口：
```Java
MyLibrary.init(context);
```
这种方法会带来一个问题，那就是 `Application` 的 `onCreate` 会堆了好多 `xxx.init`；此外如果用户忘记调用 `init`，也会出现一些问题。下面我们看看 `ProcessLifecycleOwner` 是如何解决这个问题的：
```Java
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }

    // ...
}
```
通过注册一个 `ContentProvider`，可以保证在进程启动的时候 `onCreate` 被调用，如此一来即可以实现无侵入式的初始化。当然，要承担一点点 `ContentProvider` 带来的成本，用户也没办法进行延迟初始化。

## context.getApplicationContext() 返回的是什么

我们顺着这个 `init` 调用看看 `ProcessLifecycleOwner`：
```Java
public class ProcessLifecycleOwner implements LifecycleOwner {
    static void init(Context context) {
        sInstance.attach(context);
    }

    void attach(Context context) {
        Application app = (Application) context.getApplicationContext();
        // ...
    }
}
```
androidx 里强制把 `context.getApplicationContext()` 的返回值强制转换为了 `Application`，从实用主义的角度，我们有理由相信，至少短时间内，`context.getApplicationContext()` 返回的就是 `Application` 实例。


## Android 10 的 Application.ActivityLifecycleCallbacks

我们继续往下看，会发现 `ActivityLifecycleCallbacks` 在 Android 10 多了不少新的生命周期回调：
```Java
public class ProcessLifecycleOwner implements LifecycleOwner {
    void attach(Context context) {
        // ...

        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @Override
            public void onActivityPreCreated(@NonNull Activity activity,
                    @Nullable Bundle savedInstanceState) {
                activity.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
                    @Override
                    public void onActivityPostStarted(@NonNull Activity activity) {
                        activityStarted();
                    }

                    @Override
                    public void onActivityPostResumed(@NonNull Activity activity) {
                        activityResumed();
                    }
                });
            }

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                if (Build.VERSION.SDK_INT < 29) {
                    ReportFragment.get(activity).setProcessListener(mInitializationListener);
                }
            }

            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }

            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }
}
```

我们发现，主要是多出了 Pre/Post 回调，用于在特定生命周期的前面/后面执行一些操作。后面我们还会看到，在没有系统支持的情况下，要实现这个功能还是挺折腾的。

`onActivityPreCreated` 是 Android 10 新增的接口，在 10 以下不会被调用，低版本的 `onActivityPostStarted` 和 `onActivityPostResumed` 通过一个辅助的 `ReportFragment` 来间接获取。


## 如何保证每个 activity 都有一个 ReportFragment

`ReportFragment` 有一个 `injectIfNeededIn` 静态方法，用来给 `activity` 实例注入一个 `reportFragment`：
```Java
public class ReportFragment extends Fragment {
        public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            // On API 29+, we can register for the correct Lifecycle callbacks directly
            activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
        // Prior to API 29 and to maintain compatibility with older versions of
        // ProcessLifecycleOwner (which may not be updated when lifecycle-runtime is updated and
        // need to support activities that don't extend from FragmentActivity from support lib),
        // use a framework fragment to get the correct timing of Lifecycle events
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
}
```

这里出现了一个我们不太常见的调用 `executePendingTransactions`，它可以立即执行一个 `fragmentTransaction`，这样一来，即使*连续多次调用*，在第一次执行后，紧接着的调用也能 find 到刚刚添加的实例。（如果没有 `executePendingTransactions`，会添加多个 `reportFragment`）

调用 `injectIfNeededIn` 的地方一共有三个：
1. `LifecycleDispatcher` 里注册的 `ActivityLifecycleCallbacks` 在 `onActivityCreated` 的时候调用一次
2. `androidx.activity.ComponentActivity` 和 `androidx.core.app.ComponentActivity` 的 `onCreate`；其中，前者继承了后者

由于在 `Activity.onCreate` 里最终会调用到 `ActivityLifecycleCallbacks.onActivityCreated`，这里其实我们只需要第一个。是否有重复调用问题影响不大，这里就不纠结了。


## 如何在 Android 10 以下获取 onActivityPostStarted 事件

了解了 `ReportFragment` 后，我们现在可以来看看 `onActivityPostStarted` 是如何在低于 Android 10 的系统实现的。需要说明的是，我们前面通过 `application` 注册的回调的 `onActivityStarted` 是在 `Activity.onStart` 里调用的，如果我们想要在 `onStart` 这个生命周期函数完全执行后再做一些操作，就不能直接使用 `onActivityStarted`；当然，Android 10 以后可以使用 `onActivityPostStarted`。

为了获取 `onActivityPostStarted`，需要借助前面我们提到的 `ReportFragment`，当 api level 小于 29 时，我们设置了 processListener：
```Java
public class ProcessLifecycleOwner implements LifecycleOwner {
    void attach(Context context) {
        // ...

        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            // ...

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                if (Build.VERSION.SDK_INT < 29) {
                    ReportFragment.get(activity).setProcessListener(mInitializationListener);
                }
            }

            // ...
        });
    }
}

public class ReportFragment extends Fragment {
    private ActivityInitializationListener mProcessListener;

    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }
}
```

随后，在 `fragment` 对应的生命周期回调我们就可以拿到所需的那个 `onActivityPostStarted`：
```Java
public class ReportFragment extends Fragment {
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
    }

}
```

这里也提示我们，`Fragment` 的 `onStart` 会在 `Activity` 的 `onStart` **后面**执行。此外，`onResume` 的情况也一样。至于其他生命周期函数的调用顺序，后面 lifecycle 的时候我们再研究了。


