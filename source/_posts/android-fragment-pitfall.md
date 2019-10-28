---
title: Fragment 使用陷阱
date: 2019-10-28 22:10:46
categories: Android
tags: Android
---

本篇文章主要总结过去在项目里遇到的由于 `Fragment` 误用带来的一些问题，不涉及 `Fragment` 的具体用法。这里说的 `Fragment` 带来的问题，基本是由 `Activity` 被系统销毁后自动重新创建所引发的。下面我们从 `Fragment` 的基本使用、参数传递和 `ViewPager` 的交互这三个方面来分别讨论。

## `Fragment` 的基本使用

使用 `Fragment` 的一种很常见的用法是，通过 `FragmentManager` 把一个实例添加到 view 里面：
```Java
class YourActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // ...
        FragmentManager manager = getSupportFragmentManager();
        FragmentTransaction transaction = manager.beginTransaction();
        transaction.replace(R.id.container, new YourFragment());
        transaction.commit();
    }
}
```

在继续往下看之前，请读者先想想，这段代码有什么问题？（后面基本都遵循这一个模式，先给出有问题的代码，然后说明问题，再解决它）

一开始我们就提到过，`Fragment` 带来的问题基本上都跟 `Activity` 重建有关。在这个例子中，`Activity` 重建时，系统会自动帮我们恢复 `Fragment`（`super.onCreate`），接下来我们自己又创建了一个新的实例，然后把系统创建的那个 replace 掉。表面上程序运行正常，实际上我们自己创建的那个 `Fragment` 是不必要的。正确的做法是：
```Java
FragmentManager manager = getSupportFragmentManager();
if (manager.findFragmentById(R.id.container) == null) {
    FragmentTransaction transaction = manager.beginTransaction();
    transaction.replace(R.id.container, new YourFragment());
    transaction.commit();
}
```

此外，直接把 `Fragment` 写在 xml 里面不会有这个问题，即便 `onCreate` 的时候我们总是调用 `setContentView`。我们有理由推断，这种情况下他也使用了类似的方法来防止重复创建 fragment，因为它要求我们给 `<fragment>` 加一个 id 或 tag，否则将会有一个 warning。


## 参数传递

如果是普通的类，一般我们可以这样传参数：
```Java
class SomeFragment {
    Foo mFoo;
    Bar mBar;

    void setFoo(Foo foo) { mFoo = foo; }
    void setBar(Bar bar) { mBar = bar; }
}
```
或者这样：
```Java
class SomeFragment {
    Foo mFoo;
    Bar mBar;

    SomeFragment(Foo foo, Bar bar) {
        mFoo = foo;
        mBar = bar;
    }
}
```

遗憾的是，对 `Fragment` 来说，这都是有问题的。对于第二个，还会有警告说，`Fragment` 应该有无参的构造函数。之所以要求 `Fragment` 的构造函数不带参数，是因为系统恢复它时，使用的就是无参构造函数。

和第一个例子差不多，当 `Fragment` 由系统创建的时候，`mFoo` 和 `mBar` 都会是 `null`。正确的做法应该是使用 `setArgument` 来传递：
```Java
class YourFragment extends Fragment {

    static YourFragment makeInstance(/* param0, param1, ... */) {
        YourFragment fragment = new YourFragment();
        Bundle args = new Bundle();
        // args.putXXX
        fragment.setArguments(args);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Bundle args = getArguments();
        if (args != null) {
            // unpack arguments
        }
    }
}
```

通过这种方式，即便是系统恢复 `Activity` 时自动创建的 `Fragment`，也可以 get 到原来设置的参数（不需要在 `onSaveInstanceState` 的时候保存）。当然，有些数据可能不适合放在 `Bundle` 里，这个时候可以另外用一个 `setRetainInstance(true)` 的 `fragment` 来保存（或者用 `ViewModel`）。


## 与 `ViewPager` 的交互

看下面一个例子：
```Java
public class SomePagerAdapter extends FragmentPagerAdapter {

    private static final Fragment[] mFragments = new Fragment[3];

    public SomePagerAdapter(FragmentManager fm) {
        super(fm);
    }

    @Override
    public Fragment getItem(int i) {
        if (mFragments[i] == null) {
            mFragments[i] = createFragment(i);
        }
        return mFragments[i];
    }

    private Fragment createFragment(int index) {
        switch (index) {
            case 0:
                return new Fragment1();
            case 1:
                return new Fragment2();
            case 2:
                return new Fragment3();
            default:
                throw new IllegalArgumentException();
        }
    }

    @Override
    public int getCount() {
        return mFragments.length;
    }
}
```

这一段代码的问题跟前面两个比起来隐晦得多，也不是所有这样写的代码都会出问题。某些情况下，比方说，用户触发了某个动作后，我们想获取当前的 `fragment` 并做一些操作：
```Java
void foo() {
    Fragment fragment = mPagerAdapter.getItem(mViewPager.getCurrentItem());
    fragment.bar();
}
```
我们再假设 `bar()` 方法里调用了 `getActivity()`，这样，线上就会出现当前 `fragment` 的 `getActivity` 竟然返回 `null` 的崩溃……

这个时候我们开始推断：用户触发动作（比方说按钮点击），说明 view 已经创建完成并且是可用的，这说明 fragment 肯定是 attach 过的；这样一来，`getActivity` 不可能为空。。。（此处需要一个抓狂的表情）

哎，肯定是用户的手机出了什么毛病，加个判断保护一下好了：
```Java
class Fragment1 {
    void bar() {
        Activity activity = getActivity();
        if (activity != null) {
            // ...
        }
    }
}
```
对了，别忘记给所有 `getActivity` 都加上判空。

这里说个题外话，如果 `getActivity` 真的需要在每个地方都加上判空，那么在 `Fragment` 中定义的时候，就应该加上 `@Nullable`。只是在很多情况下，我们使用它的时候是明确知道不会为空的（比如我们例子中的情形），此时加上判空只会让代码更难理解。这应该也是它没有加 `@Nullable` 的原因。

如果你厌倦了给每个 `getActivity` 加判空，下面我们来真正解决他。和文章开头说的一样，遇到这些没走生命周期的 fragment，首先我们就需要怀疑是不是发生了 activity 的重建。按照前面我们说的，如果 activity 重建，系统会自动帮我们恢复所有的 fragment。这样一来，adapter 就不应该再创建 fragment 了。

我们可以通过 `FragmentPagerAdapter` 的 `instantiateItem` 来确认这一点：
```Java
public abstract class FragmentPagerAdapter extends PagerAdapter {

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }
}
```

可以看到，在调用 `getItem` 前，它会先 `findFragmentByTag`。显然，在 activity 重建的情况下，`findFragmentByTag` 会返回系统创建的那个 fragment。而后，在上面的例子中，当用户触发了 `bar` 的执行时，我们调用 `getItem` 创建了一个新的 fragment，这个新创建的 fragment 并没有 attach 到 activity，所以它的 `getActivity` 返回 `null` 也就不奇怪了。

现在，我们有足够的理由说，adapter 的最佳实践应该是， `getItem` 总返回一个新创建的 fragment（如果它叫 makeItem、createItem，就不会有这么多麻烦事了）：
```Java
public class SomePagerAdapter extends FragmentPagerAdapter {

    private static final Fragment[] mFragments = new Fragment[3];

    public SomePagerAdapter(FragmentManager fm) {
        super(fm);
    }

    @NonNull
    @Override
    public Object instantiateItem(@NonNull ViewGroup container, int position) {
        Object object = super.instantiateItem(container, position);
        mFragments[position] = (Fragment) object;
        return object;
    }

    @Nullable
    public Fragment getFragment(int position) {
        return mFragments[position];
    }

    @Override
    public Fragment getItem(int i) {
        switch (i) {
            case 0:
                return new Fragment1();
            case 1:
                return new Fragment2();
            case 2:
                return new Fragment3();
            default:
                throw new IllegalArgumentException();
        }
    }

    @Override
    public int getCount() {
        return mFragments.length;
    }
}
```
作为一个妥协，我们把父类 `instantiateItem` 返回的 fragment 缓存了起来。

