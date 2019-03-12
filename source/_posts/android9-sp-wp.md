---
title: Android P 源码分析 2 - 强弱指针的实现
date: 2019-03-12 09:01:01
categories: Android
tags: [Android source]
---

继上篇我们学习了 Android 轻量级指针的实现，是时候来看“重量级”指针的实现了。在 Android 里，“重量级”指针指的是 `RefBase` 和 `sp/wp` 配合使用的情况，它提供了完整的强、弱指针的支持。

<!-- more -->

考虑这样一种情况，A 持有 B，B 持有 A，C 持有 A。如果只使用简单的引用计数，在 C 释放 A 后，A、B 各自的计数值都为 1，永远不会被销毁，也无法再访问。这就是经典的循环引用问题。

引入弱指针后，我们可以让 A 持有 B 的强指针，而 B 指持有 A 的弱指针。这样一来，在 C 释放 A 后，A 的引用计数将降为 0 从而被销魂；A 销毁的同时，A 所持有的 B 的强指针也会销毁，于是 B 的引用计数也降为 0，B 被销毁。

了解了这些基础知识后，下面我们就来看看 `RefBase` 的源码

## RefBase

### 类定义

这里我先把 `RefBase` 的定义摆上来，后面我们捡最重要的几个看。

```C++
// system/core/libutils/include/utils/RefBase.h
class RefBase
{
public:
            void            incStrong(const void* id) const;
            void            decStrong(const void* id) const;
    
            void            forceIncStrong(const void* id) const;

            //! DEBUGGING ONLY: Get current strong ref count.
            int32_t         getStrongCount() const;

    class weakref_type
    {
    public:
        RefBase*            refBase() const;

        void                incWeak(const void* id);
        void                decWeak(const void* id);

        // acquires a strong reference if there is already one.
        bool                attemptIncStrong(const void* id);

        // acquires a weak reference if there is already one.
        // This is not always safe. see ProcessState.cpp and BpBinder.cpp
        // for proper use.
        bool                attemptIncWeak(const void* id);

        //! DEBUGGING ONLY: Get current weak ref count.
        int32_t             getWeakCount() const;

        //! DEBUGGING ONLY: Print references held on object.
        void                printRefs() const;

        //! DEBUGGING ONLY: Enable tracking for this object.
        // enable -- enable/disable tracking
        // retain -- when tracking is enable, if true, then we save a stack trace
        //           for each reference and dereference; when retain == false, we
        //           match up references and dereferences and keep only the
        //           outstanding ones.

        void                trackMe(bool enable, bool retain);
    };

            weakref_type*   createWeak(const void* id) const;
            
            weakref_type*   getWeakRefs() const;

            //! DEBUGGING ONLY: Print references held on object.
    inline  void            printRefs() const { getWeakRefs()->printRefs(); }

            //! DEBUGGING ONLY: Enable tracking of object.
    inline  void            trackMe(bool enable, bool retain)
    { 
        getWeakRefs()->trackMe(enable, retain); 
    }

    typedef RefBase basetype;

protected:
                            RefBase();
    virtual                 ~RefBase();
    
    //! Flags for extendObjectLifetime()
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
    
            void            extendObjectLifetime(int32_t mode);
            
    //! Flags for onIncStrongAttempted()
    enum {
        FIRST_INC_STRONG = 0x0001
    };
    
    // Invoked after creation of initial strong pointer/reference.
    virtual void            onFirstRef();
    // Invoked when either the last strong reference goes away, or we need to undo
    // the effect of an unnecessary onIncStrongAttempted.
    virtual void            onLastStrongRef(const void* id);
    // Only called in OBJECT_LIFETIME_WEAK case.  Returns true if OK to promote to
    // strong reference. May have side effects if it returns true.
    // The first flags argument is always FIRST_INC_STRONG.
    // TODO: Remove initial flag argument.
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);
    // Invoked in the OBJECT_LIFETIME_WEAK case when the last reference of either
    // kind goes away.  Unused.
    // TODO: Remove.
    virtual void            onLastWeakRef(const void* id);

private:
    friend class weakref_type;
    class weakref_impl;
    
                            RefBase(const RefBase& o);
            RefBase&        operator=(const RefBase& o);

private:
        weakref_impl* const mRefs;
};
```

在看他的函数的实现前，我们先对比一下 `LightRefBase`，看看他们两个有什么不同。首先最明显的是，`RefBase` 不是一个类模板，这个我们在上篇已经提到了；也因此 `RefBase::~RefBase` 是一个虚函数。其次，`RefBase` 还定义了一个 `weakref_type`，他的真正实现是 `weakref_impl`，所有的引用计数都记录在 `weakref_impl` 里。

好事者这个时候就要问了，为什么 `RefBase` 不像 `LightRefBase` 把引用计数直接用成员变量来存储？（人家这么写，肯定是有理由的呀）这里的关键就在弱指针上。当用户持有弱指针的时候，需要提供一种途径，让他尝试转换成强指针。如果把引用计数等信息都存放在 `RefBase` 里，当对象已经销毁但有弱指针指向它的时候，弱指针就没有信息可以判断是否能够升级为强指针了。现在我们把引用计数都放在 `weakref_impl` 里，`RefBase` 对象可以先销毁；只要有 `weakref_imp` 在，`wp` 就能够根据 `weakref_impl` 中的信息判断是否能够提升为 `sp`。

接下来我们看看 `weakref_impl`：
```C++
// system/core/libutils/RefBase.cpp
#define INITIAL_STRONG_VALUE (1<<28)

class RefBase::weakref_impl : public RefBase::weakref_type
{
public:
    std::atomic<int32_t>    mStrong;
    std::atomic<int32_t>    mWeak;
    RefBase* const          mBase;
    std::atomic<int32_t>    mFlags;

#if !DEBUG_REFS

    explicit weakref_impl(RefBase* base)
        : mStrong(INITIAL_STRONG_VALUE)
        , mWeak(0)
        , mBase(base)
        , mFlags(0)
    {
    }

    void addStrongRef(const void* /*id*/) { }
    void removeStrongRef(const void* /*id*/) { }
    void renameStrongRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void addWeakRef(const void* /*id*/) { }
    void removeWeakRef(const void* /*id*/) { }
    void renameWeakRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void printRefs() const { }
    void trackMe(bool, bool) { }
#else
    // ...
#endif
};
```
`weakref_impl` 继承了 `weakref_type`，在非 debug 模式下，他的另外一些成员函数都是空实现，我们直接忽略。`mFlags` 的取值是 `RefBase` 中定义的 `enum`：
```C++
//! Flags for extendObjectLifetime()
enum {
    OBJECT_LIFETIME_STRONG  = 0x0000,
    OBJECT_LIFETIME_WEAK    = 0x0001,
    OBJECT_LIFETIME_MASK    = 0x0001
};
```
创建 `weakref_impl` 时，使用的是默认的 lifetime `OBJECT_LIFETIME_STRONG`。

有了总体的认识后，接下来我们开始看源码，探究一些小细节。

## 强指针的实现

在这一小节我们先来看强指针的实现，弱指针留到后面。上一节我们讲 `LightRefBase` 的时候已经知道，`sp` 在创建的时候会调用对象的 `incStrong`，销毁的时候调用 `decStrong`。下面我们就来看看 `RefBase` 这两个函数的实现。

首先是对象的创建：
```C++
// system/core/libutils/RefBase.cpp
RefBase::RefBase()
    : mRefs(new weakref_impl(this)) {}
```
嗯，是比较单调无趣。下面我们看看 `incStrong`：
```C++
// system/core/libutils/RefBase.cpp
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->incWeak(id);
    
    refs->addStrongRef(id);
    const int32_t c = refs->mStrong.fetch_add(1, std::memory_order_relaxed);
    ALOG_ASSERT(c > 0, "incStrong() called on %p after last strong ref", refs);
    if (c != INITIAL_STRONG_VALUE)  {
        return;
    }

    int32_t old __unused = refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE, std::memory_order_relaxed);
    // A decStrong() must still happen after us.
    ALOG_ASSERT(old > INITIAL_STRONG_VALUE, "0x%x too small", old);
    refs->mBase->onFirstRef();
}
```
`weakref_impl` 没有定义 `incWeak` 函数，这里实际调用的是他的父类 `weakref_type` 的 `incWeak`：
```C++
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->addWeakRef(id);  // 空实现
    const int32_t c __unused = impl->mWeak.fetch_add(1,
            std::memory_order_relaxed);
    ALOG_ASSERT(c >= 0, "incWeak called on %p after last weak ref", this);
}
```
因为实际的实现就是 `weakref_impl`，所以这里的强制类型转换不会出错。跟 `LightRefBase` 的情况一样，由于计数值跟其他的数据没有什么依赖，这里用 `memory_order_relaxed` 就可以了。

总的来说，`incStrong` 所做的就是给 `weakref_impl` 的 `mStrong` 和 `mWeak` 都加 1。

接下来我们看看 `decStrong`：
```C++
// system/core/libutils/RefBase.cpp
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id);  // 空实现
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release);
    LOG_ALWAYS_FATAL_IF(BAD_STRONG(c), "decStrong() called on %p too many times",
            refs);
    if (c == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;
            // The destructor does not delete refs in this case.
        }
    }
    // Note that even with only strong reference operations, the thread
    // deallocating this may not be the same as the thread deallocating refs.
    // That's OK: all accesses to this happen before its deletion here,
    // and all accesses to refs happen before its deletion in the final decWeak.
    // The destructor can safely access mRefs because either it's deleting
    // mRefs itself, or it's running entirely before the final mWeak decrement.
    //
    // Since we're doing atomic loads of `flags`, the static analyzer assumes
    // they can change between `delete this;` and `refs->decWeak(id);`. This is
    // not the case. The analyzer may become more okay with this patten when
    // https://bugs.llvm.org/show_bug.cgi?id=34365 gets resolved. NOLINTNEXTLINE
    refs->decWeak(id);
}
```
这里就直接了当地多了，直接修改 `weakref_impl` 的成员变量。

根据上篇文章我们对 `sp` 的了解，只有在准备销魂对象的时候才会调用 `decStrong`。这里 `fetch_sub` 使用 `memory_order_release` 就保证了接下来我们要销毁对象时，前面对对象的操作都已经执行完（release 操作相当于在前面放了一个内存屏障，确保前面的操作不会被重排序到 `fetch_sub` 的后面）。

如果 `c == 1`，说明我们是最后一个引用对象的人，接下来就可以准备删除对象了。这里 `atomic_thread_fense` 的使用跟前面 `LightRefBase` 的用法是一样的。

如果 `mFlag` 是 `OBJECT_LIFETIME_STRONG`，表示对象的生命周期由强指针控制，当强引用计数值降为 0 的时候，就需要删除对象。

由于我们在 `incStrong` 里增加了弱引用计数，这里也要 `decWeak`。


## 弱指针的实现

```C++
// system/core/libutils/include/utils/RefBase.h
template <typename T>
class wp
{
public:
    typedef typename RefBase::weakref_type weakref_type;

    wp(const wp<T>& other);
    explicit wp(const sp<T>& other);

    ~wp();

    // promotion to sp
    sp<T> promote() const;

    // 略去了跟我们关注的主题不相关的其他一大堆函数

private:
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;

    T*              m_ptr;
    weakref_type*   m_refs;
};
```
`wp` 的 `m_ptr` 指向真正的对象，但这个对象可能已经被销毁了；在使用的时候需要先调用 `promote()` 提升到 `sp`，以确保对象不被销毁。`m_refs` 指针一定是有效的。

### 弱指针的创建

创建 `wp` 的代码很简单，就是把 `weakref_impl` 的 `mWeak` 计数加 1：
```C++
template<typename T>
wp<T>::wp(const wp<T>& other)
    : m_ptr(other.m_ptr), m_refs(other.m_refs)
{
    if (m_ptr) m_refs->incWeak(this);
}

template<typename T>
wp<T>::wp(const sp<T>& other)
    : m_ptr(other.m_ptr)
{
    if (m_ptr) {
        m_refs = m_ptr->createWeak(this);
    }
}
```

### 从弱指针提升至强指针

下面我们看从 `wp` 转 `sp` 的情况：
```C++
template<typename T>
sp<T> wp<T>::promote() const
{
    sp<T> result;
    if (m_ptr && m_refs->attemptIncStrong(&result)) {
        result.set_pointer(m_ptr);
    }
    return result;
}
```
这段代码并没有做太多事，关键的实现还是在 `m_refs->attemptIncStrong()`。`m_refs` 是一个 `RefBase::weakref_type*`，执行的是下面这段代码：
```C++
bool RefBase::weakref_type::attemptIncStrong(const void* id)
{
    incWeak(id);
    
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    int32_t curCount = impl->mStrong.load(std::memory_order_relaxed);

    ALOG_ASSERT(curCount >= 0,
            "attemptIncStrong called on %p after underflow", this);

    while (curCount > 0 && curCount != INITIAL_STRONG_VALUE) {
        // 情况1：
        // we're in the easy/common case of promoting a weak-reference
        // from an existing strong reference.
        if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                std::memory_order_relaxed)) {
            break;
        }
        // the strong count has changed on us, we need to re-assert our
        // situation. curCount was updated by compare_exchange_weak.
    }
    
    if (curCount <= 0 || curCount == INITIAL_STRONG_VALUE) {
        // we're now in the harder case of either:
        // - there never was a strong reference on us
        // - or, all strong references have been released
        int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            // this object has a "normal" life-time, i.e.: it gets destroyed
            // when the last strong reference goes away
            if (curCount <= 0) {
                // 情况2：
                // the last strong-reference got released, the object cannot
                // be revived.
                decWeak(id);
                return false;
            }

            // 情况3：
            // here, curCount == INITIAL_STRONG_VALUE, which means
            // there never was a strong-reference, so we can try to
            // promote this object; we need to do that atomically.
            while (curCount > 0) {
                if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                        std::memory_order_relaxed)) {
                    break;
                }
                // the strong count has changed on us, we need to re-assert our
                // situation (e.g.: another thread has inc/decStrong'ed us)
                // curCount has been updated.
            }

            if (curCount <= 0) {
                // promote() failed, some other thread destroyed us in the
                // meantime (i.e.: strong count reached zero).
                decWeak(id);
                return false;
            }
        } else {
            // 情况4： 
            // this object has an "extended" life-time, i.e.: it can be
            // revived from a weak-reference only.
            // Ask the object's implementation if it agrees to be revived
            if (!impl->mBase->onIncStrongAttempted(FIRST_INC_STRONG, id)) {
                // it didn't so give-up.
                decWeak(id);
                return false;
            }
            // grab a strong-reference, which is always safe due to the
            // extended life-time.
            curCount = impl->mStrong.fetch_add(1, std::memory_order_relaxed);
            // If the strong reference count has already been incremented by
            // someone else, the implementor of onIncStrongAttempted() is holding
            // an unneeded reference.  So call onLastStrongRef() here to remove it.
            // (No, this is not pretty.)  Note that we MUST NOT do this if we
            // are in fact acquiring the first reference.
            if (curCount != 0 && curCount != INITIAL_STRONG_VALUE) {
                impl->mBase->onLastStrongRef(id);
            }
        }
    }
    
    impl->addStrongRef(id);

#if PRINT_REFS
    ALOGD("attemptIncStrong of %p from %p: cnt=%d\n", this, id, curCount);
#endif

    // curCount is the value of mStrong before we incremented it.
    // Now we need to fix-up the count if it was INITIAL_STRONG_VALUE.
    // This must be done safely, i.e.: handle the case where several threads
    // were here in attemptIncStrong().
    // curCount > INITIAL_STRONG_VALUE is OK, and can happen if we're doing
    // this in the middle of another incStrong.  The subtraction is handled
    // by the thread that started with INITIAL_STRONG_VALUE.
    if (curCount == INITIAL_STRONG_VALUE) {
        impl->mStrong.fetch_sub(INITIAL_STRONG_VALUE,
                std::memory_order_relaxed);
    }

    return true;
}
```

每次增加强引用计数前，都要先递增一个弱引用计数。由于我们是通过弱指针来执行这个操作，`incWeak` 总是有效的，这里直接执行就可以。

前面我们说过，`weakref_type` 的实际实现是 `weakref_impl`，所以接下来的强制类型转换也是合法的。后面的操作我们分几种情况来考虑。

1. 有强指针指向这个对象。此时 `curCount > 0 && curCount != INITIAL_STRONG_VALUE`，我们在接下来的循环里尝试递增一个强引用计数。之所以要用 compare exchange，是因为在我们准备给 `mStrong` 加 1 的同时，其他线程可能要给他减 1。
   按照 C++ 文档的描述，相对于 `compare_exchange_strong`，`compare_exchange_weak` 偶尔会发生假性的（spurious）失败，但它能够在弱一致性保证的机器上提供更好的性能（由于我们是在一个循环里面执行，所以即使发生了假性失败，也会重新执行）。为了更好的性能，这里使用的是后者。
2. `curCount < 0`。在这种情况下，如果对象的生命周期是 `OBJECT_LIFETIME_STRONG`，说明对象受强引用计数控制，此时对象已经销毁。
3. `curCount == INITIAL_STRONG_VALUE`，对象创建以后还没有被强指针引用过，说明对象还存活着，此时我们在一个 `while` 循环里尝试递增 `mStrong`
4. 这种情况下，`mFlags == OBJECT_LIFETIME_WEAK`，对象的生命周期受弱引用计数控制。既然我们准备将一个弱引用提升为强引用，此时对象的弱引用计数肯定是不为 0 的。这个时候如果对象允许（`onIncStrongAttempted` 默认的实现就是返回 `true`），我们直接递增强引用计数就可以了。


### 弱指针的销毁

回顾一下强指针的内容，我们知道，`sp` 销毁的时候，会调用对象的 `decStrong`。`RefBase::decStrong` 我们在前面已经看过他的实现了，总结起来就是：
1. 如果对象受强引用计数控制（`OBJECT_LIFETIME_STRONG`），`decStrong` 在强引用计数为 0 的时候销毁 `this`（不销毁 `weakref_impl`）
2. 如果受弱引用计数控制，在强引用计数为 0 的时候不直接销毁对象。

对强指针来说，无论是上面的哪一种情况，最后都会执行 `decWeak`（强指针会使 `mStrong`、`mWeak` 都加 1）。对弱指针来说，他只会递增弱引用计数 `mWeak`，所以在销毁的时候只需要执行 `decWeak`：
```C++
// system/core/libutils/include/utils/RefBase.h
template<typename T>
wp<T>::~wp()
{
    if (m_ptr) m_refs->decWeak(this);
}
```

下面我们看看 `decWeak`:
```C++
// system/core/libutils/RefBase.cpp
void RefBase::weakref_type::decWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->removeWeakRef(id);
    const int32_t c = impl->mWeak.fetch_sub(1, std::memory_order_release);
    LOG_ALWAYS_FATAL_IF(BAD_WEAK(c), "decWeak called on %p too many times",
            this);
    if (c != 1) return;
    atomic_thread_fence(std::memory_order_acquire);

    int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
    if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
        // This is the regular lifetime case. The object is destroyed
        // when the last strong reference goes away. Since weakref_impl
        // outlives the object, it is not destroyed in the dtor, and
        // we'll have to do it here.
        if (impl->mStrong.load(std::memory_order_relaxed)
                == INITIAL_STRONG_VALUE) {
            // Decrementing a weak count to zero when object never had a strong
            // reference.  We assume it acquired a weak reference early, e.g.
            // in the constructor, and will eventually be properly destroyed,
            // usually via incrementing and decrementing the strong count.
            // Thus we no longer do anything here.  We log this case, since it
            // seems to be extremely rare, and should not normally occur. We
            // used to deallocate mBase here, so this may now indicate a leak.
            ALOGW("RefBase: Object at %p lost last weak reference "
                    "before it had a strong reference", impl->mBase);
        } else {
            // ALOGV("Freeing refs %p of old RefBase %p\n", this, impl->mBase);
            delete impl;
        }
    } else {
        // This is the OBJECT_LIFETIME_WEAK case. The last weak-reference
        // is gone, we can destroy the object.
        impl->mBase->onLastWeakRef(id);
        delete impl->mBase;
    }
}
```
这里 `atomic_thread_fense` 的用法跟 `decStrong` 里是一样的。在弱引用计数降为 0 的时候，分 3 中情况处理：
1. `OBJECT_LIFETIME_STRONG`，但是没有强指针引用过这个对象。这个只有在我们使用弱指针引用对象但却从没把弱指针提升到强指针（也就是说，我们根本没使用过这个对象）的情况下才会发生。由于很可能是个错误，这个写了个 warning 日志。
2. `OBJECT_LIFETIME_STRONG`，有强指针引用过这个对象，这是最正常的情况。此时不存在任何的强、弱指针指向对象，所以把 `weakref_impl` 也删掉。
3. `OBJECT_LIFETIME_WEAK`，由于对象受弱引用计数控制而此时弱引用计数为 0，所以需要删除对象（`delete impl->mBase`）。

无论是 `decStrong` 还是 `decWeak`，删除对象后，最终都会执行到 `RefBase::~RefBase`：
```C++
RefBase::~RefBase()
{
    int32_t flags = mRefs->mFlags.load(std::memory_order_relaxed);
    // Life-time of this object is extended to WEAK, in
    // which case weakref_impl doesn't out-live the object and we
    // can free it now.
    if ((flags & OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) {
        // It's possible that the weak count is not 0 if the object
        // re-acquired a weak reference in its destructor
        if (mRefs->mWeak.load(std::memory_order_relaxed) == 0) {
            delete mRefs;
        }
    } else if (mRefs->mStrong.load(std::memory_order_relaxed)
            == INITIAL_STRONG_VALUE) {
        // We never acquired a strong reference on this object.
        LOG_ALWAYS_FATAL_IF(mRefs->mWeak.load() != 0,
                "RefBase: Explicit destruction with non-zero weak "
                "reference count");
        // TODO: Always report if we get here. Currently MediaMetadataRetriever
        // C++ objects are inconsistently managed and sometimes get here.
        // There may be other cases, but we believe they should all be fixed.
        delete mRefs;
    }
    // For debugging purposes, clear mRefs.  Ineffective against outstanding wp's.
    const_cast<weakref_impl*&>(mRefs) = NULL;
}
```
如果对象受弱引用计数控制，在 `decWeak` 中我们删除对象，而后 `weakref_impl` 在 `~RefBase` 中删除。

前面 `decWeak` 的第一种情况下，我们只是写了个日志然后什么也没做。如果受强引用计数控制但却没有被强指针引用过，并且执行了 `~RefBase`，说明对象要么是被手动 `delete`，要么是在栈上分配的。不管哪种情况，此时都应该释放 `mRefs`，否则将会有内存泄漏。

最后做个总结。在正常的情况下，
1. 如果对象的生命周期是 `OBJECT_LIFETIME_STRONG`，这也是默认的情况。在 `decStrong` 中，如果强引用计数为 0，将删除对象；在 `decWeak` 中，如果弱引用计数为 0，将删除 `weakref_impl`
2. 如果对象的生命周期是 `OBJECT_LIFETIME_WEAK`，对象将在 `decWeak` 中删除，`weakref_impl` 则是在 `~RefBase` 中回收的。




