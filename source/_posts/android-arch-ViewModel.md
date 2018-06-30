---
title: Android arch components 源码分析（1）—— ViewModel
date: 2018-06-30 13:26:59
categories: Android
tags: Android arch
---

本篇主要关注 `ViewModel` 的实现而非起用法，关于他的用法，可以参考[这里](https://developer.android.google.cn/topic/libraries/architecture/)。

`ViewModel` 主要用于在 activity/fragment 被自动销毁时保存一些数据。从实现原理上讲，主要就是利用了 `fragment.setRetainInstance(true)`。如此一来，这个 `fragment` 就能够跨越 `activity` 的生命周期。


## 总览

- `ViewModel`：这个是我们的主角。我们定义的 model 类需要继承它。
- `ViewModelProvider`：用于生成 `ViewModel` 实例
- `ViewModelStore`：跟他的名字一样，主要用来存储 `ViewModel` 的实例。
- `HolderFragment`：这个就是我们上面提到的调用了 `setRetainInstance(true)` 的那个 `fragment`。`ViewModelStore` 实例存储会在这里。

下面是 Google 给出的[示例](https://github.com/googlecodelabs/android-lifecycles) step2 中的一段代码：
```Java
ChronometerViewModel chronometerViewModel
       = ViewModelProviders.of(this).get(ChronometerViewModel.class);
```

下面我们就根据这个调用来学习 `ViewModel` 的源码。

## ViewModel

前面我们说过，自己定义的 model 类需要继承它。这里借花献佛，我们直接看 Google 的 sample：
```Java
public class ChronometerViewModel extends ViewModel {
    // 不用关心它的内容
}
```

`ViewModel` 虽然是主角，但他非常的简单：
```Java
public abstract class ViewModel {
    protected void onCleared() {
    }
}
```
就这样，他只是定义了一个空方法 `onCleared()`。当对应的 model 实例被销毁时，`onCleared()` 讲会执行。通过让他成为 `abstract class` 并给予 `onCleared` 一个默认实现，让 `ViewModel` 有了 tag interface 的效果。

> 所谓的 tag interface 是指不带任何方法的 `interface`。

接下来是男二号 `AndroidViewModel`：
```Java
public class AndroidViewModel extends ViewModel {
    private Application mApplication;

    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }

    @NonNull
    public <T extends Application> T getApplication() {
        return (T) mApplication;
    }
}
```

没有太多可圈可点的东西，我们继续看下一个。


## ViewModelProvider

```Java
ChronometerViewModel chronometerViewModel
       = ViewModelProviders.of(this).get(ChronometerViewModel.class);
```
我们继续从这个调用往下看。

`ViewModelProviders` 可以看成是 `ViewModelProvider` 的工厂或相关工具类的合集。他的命名跟 JDK 里的 `Collections/Arrays` 类似。这里的 `this` 是 `FragmentActivity`，所以接下来执行的是：
```Java
public class ViewModelProviders {
    @NonNull
    @MainThread
    public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        return of(activity, null);
    }

    @NonNull
    @MainThread
    public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(ViewModelStores.of(activity), factory);
    }
}
```

可以看到，最终 `ViewModelProviders` 会创建一个 `ViewModelProvider` 实例并返回。

`Factory` 是用于创建 model 实例的工厂：
```Java
public interface Factory {
    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```
默认的实现是 `AndroidViewModelFactory`：
```Java
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

    private static AndroidViewModelFactory sInstance;

    @NonNull
    public static AndroidViewModelFactory getInstance(@NonNull Application application) {
        if (sInstance == null) {
            sInstance = new AndroidViewModelFactory(application);
        }
        return sInstance;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            // model 类继承了 AndroidViewModel
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
        // 否则，由父类 `NewInstanceFactory` 处理
        return super.create(modelClass);
    }
}
```

父类 `NewInstanceFactory` 的实现一样很简单：
```Java
public static class NewInstanceFactory implements Factory {

    @SuppressWarnings("ClassNewInstance")
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        try {
            return modelClass.newInstance();
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
}
```

从这里可以看出，使用默认的 `Factory` 实现时，如果 model 类继承 `ViewModel`，需要有一个默认构造函数；如果继承 `AndroidViewModel`，必须有一个以 `Application` 为唯一参数构造函数。否则，我们需要自己实现一个 `Factory`。

我们先把 `ViewModelStore` 放一放，先假设成功拿到了他的实例，于是，我们创建 `ViewModelProvider` 实例：
```Java
public static ViewModelProvider of(@NonNull FragmentActivity activity,
                                   @Nullable Factory factory) {
    // ...
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}

public class ViewModelProvider {

    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }

    // ...
}
```

通过 `ViewModelProvider` 获取 model 实例时，使用的是他的 `get` 方法：
```
public class ViewModelProvider {

    private static final String DEFAULT_KEY =
            "android.arch.lifecycle.ViewModelProvider.DefaultKey";

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        // 当使用不同的类加载起加载同一个类的时候，这里会是 false
        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }

        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
}

```

`ViewModelProvider` 他的职责就是从 `ViewModelStore` 里取出对象，如果对象不存在，就新创建一个，并把新创建的这个对象放到 `ViewModelStore`。


## ViewModelStore

和 `ViewModelProvider` 一样，`ViewModelStore` 也有一个工厂类叫 `ViewModelStores`。
```Java
public class ViewModelStores {
    @NonNull
    @MainThread
    public static ViewModelStore of(@NonNull FragmentActivity activity) {
        // 我们的 activity 可以自己实现 ViewModelStoreOwner
        // 默认情况下，这里的判断为 false，ViewModelStoreOwner 由 HolderFragment 实现
        if (activity instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) activity).getViewModelStore();
        }
        // holderFragmentFor 在 HolderFragment 中实现，我们留到下一节再看
        return holderFragmentFor(activity).getViewModelStore();
    }
}


public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```

`HolderFragment` 实现了 `ViewModelStoreOwner` 接口，`holderFragmentFor(activity)` 返回 `activity` 对应的 `holderFragment` 后，即可以拿到 `ViewModelStore` 实例。

```Java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}
```


## HolderFragment

`HolderFragment` 是整个实现的核心。
```Java
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    // ...

    public HolderFragment() {
        // 如此一来，当 activity 由于屏幕旋转等被系统销毁时，这个 fragment 实例也不会被销毁
        setRetainInstance(true);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 把 activity 从 mNotCommittedActivityHolders 中移除
        sHolderFragmentManager.holderFragmentCreated(this);
    }

    public static HolderFragment holderFragmentFor(FragmentActivity activity) {
        return sHolderFragmentManager.holderFragmentFor(activity);
    }

    static class HolderFragmentManager {
        private Map<Activity, HolderFragment> mNotCommittedActivityHolders = new HashMap<>();

        private ActivityLifecycleCallbacks mActivityCallbacks =
                new EmptyActivityLifecycleCallbacks() {
                    @Override
                    public void onActivityDestroyed(Activity activity) {
                        HolderFragment fragment = mNotCommittedActivityHolders.remove(activity);
                        // fragment 创建成功后，会把 activity 从 mNotCommittedActivityHolders 中
                        // 移除。如果 fragment != null，说明 fragment 没有创建完 activity 就跪了
                        if (fragment != null) {
                            Log.e(LOG_TAG, "Failed to save a ViewModel for " + activity);
                        }
                    }
                };

        private boolean mActivityCallbacksIsAdded = false;


        HolderFragment holderFragmentFor(FragmentActivity activity) {
            FragmentManager fm = activity.getSupportFragmentManager();
            // 通过 fragmentManager 获取 fragment 实例
            HolderFragment holder = findHolderFragment(fm);
            if (holder != null) {
                return holder;
            }
            holder = mNotCommittedActivityHolders.get(activity);
            if (holder != null) {
                return holder;
            }

            if (!mActivityCallbacksIsAdded) {
                mActivityCallbacksIsAdded = true;
                activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
            }
            holder = createHolderFragment(fm);
            // 我们 add 进去的 fragment 并不会马上就执行完（也就是说，这个方法执行完成后，马上再
            // 调用一次，上面的 findHolderFragment 会返回 null。但是这没有关系，因为接下来我们还可
            // 从 mNotCommittedActivityHolders 获取到对应的实例），所以我们这里先把他放在
            // mNotCommittedActivityHolders 中。Not Committed 表示 fragment 的 commit 还没有完成
            mNotCommittedActivityHolders.put(activity, holder);
            return holder;
        }

        private static HolderFragment createHolderFragment(FragmentManager fragmentManager) {
            HolderFragment holder = new HolderFragment();
            // 这个 fragment 只是用来存数据，允许他的状态丢失可以让用户在更多情景下使用我们的API
            // 例如，onStop() 中也可以使用（当然，onDestroy 就不行了，因为我们需要往 activity 悄悄
            // 添加一个 fragment）
            fragmentManager.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();
            return holder;
        }
    }
}
```

`HolderFragment` 对 `ViewModelStoreOwner` 实现是相当直接的：
```Java
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    private ViewModelStore mViewModelStore = new ViewModelStore();

    @Override
    public void onDestroy() {
        super.onDestroy();
        mViewModelStore.clear();
    }

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }
}
```

到这里，`ViewModel` 的实现我们就看完了。需要注意的是，`ViewModel` 还支持 `fragment`，这部分跟 `activity ` 是类似的，有兴趣的读者自己看一看就好。
