---
title: Android P 源码分析 1 - 轻量级智能指针的实现
date: 2019-03-06 19:23:28
categories: Android
tags: [Android source]
description: 作为 Android 源码分析系列文章的第一篇，我们先看 LightRefBase 的源码，热热身。
---

## 开篇词

去年（2018）二季度写过几篇 Android 源码相关的文章，后来由于太懒中断了，一晃眼一整年什么也没干成。经过几个月的迷茫，终于在年底开始发奋学习。慢慢把一些基础捡回来后，兜兜转转，看源码的时机又来了。文章标题里的那个“1”显然表示此刻的我雄心勃勃，也希望自己能够坚持下去，改掉虎头蛇尾的毛病。

分析 Android 源码的书籍中，最厚重的无疑是老罗的《Android 源代码情景分析》，目前我也是使用它作为主要的参考书。前期的写作基本会按照老罗书中的脉络进行，如果有幸坚持下来，再另外找话题继续探索。跟老罗书中不同的是，喜新厌旧的我将会基于 Android P（`pie-release`分支）来讲解，不然就太没意思了。

由于作者水平有限，没办法在文章把涉及的知识点给大家一一罗列，只能在相关地方推荐几本参考书。如果大家对某个知识点有疑惑，可以是找个安静的时间，慢慢享受一本纸质书。

下面我们进入正题，先拿 Android 的轻量级指针来热热身。

## 关于智能指针的一点点背景知识

文章假设你有一定的 C++ 基础，不熟悉的读者可以参考《C++ Primer》。也希望读者可以下载一份源码，毕竟在网页上没法在代码直接进行跳转，整体性也差一点。如果你想了解更多智能指针的知识，《More Effective》将会是一本很棒的书，很值得一读。

标准的 C++ 并支持垃圾收集，这就需要用户手动释放内存资源。当多个模块通过一个指针共享对象实例时，对象的所有权往往非常地模糊，这就很容易让用户在什么时候删除对象这个问题上产生疑惑。幸运的时候，利用 RAII(Resource Acquisition Is Initialization，参考《The C++ Programming Language》，不了解也没关系)，我们可以系统地、自动地管理对象的生命周期。

所谓的智能指针，就是在对象的内部维持一个计数；我们在类的构造函数里对计数增加 1，并在析构函数减 1。如果当前我们是最后一个人引用这个对象，那么计数值在析构函数里减 1 后就应该等于 0，此时对象可以被安全地删除。

## 轻量级指针的用法

先了解一下 API，对我们阅读源码是非常有帮助的，这里我们先看看一个小例子，学一学怎么使用轻量级指针。

首先，对应希望被智能指针引用的类应该继承 `LightRefBase`:
```C++
class Foo : public LightRefBase<Foo> {
 public:
  void foo() {}
};
```
接下来，我们通过 `sp` 来引用它：
```C++
void bar() {
  // 创建一个 sp，sp 指 strong pointer，它是一个智能指针，用来管理我们的 Foo 对象
  sp<Foo> p{new Foo};   // 此时计数值为 1
  {
    auto p2 = p;    // 计数值 = 2
    p2->foo();
  } // p2 销毁，计数值 = 1
  p->foo();
}   // p 销毁，计数值 = 0，Foo 对象也销毁
```

下面我们一起来看看它的源码。

## 计数器 LightRefBase 的实现

```C++
// system/core/libutils/include/utils/LightRefBase.h
template <class T>
class LightRefBase
{
public:
    inline LightRefBase() : mCount(0) { }
    inline void incStrong(__attribute__((unused)) const void* id) const {
        mCount.fetch_add(1, std::memory_order_relaxed);
    }
    inline void decStrong(__attribute__((unused)) const void* id) const {
        if (mCount.fetch_sub(1, std::memory_order_release) == 1) {
            std::atomic_thread_fence(std::memory_order_acquire);
            delete static_cast<const T*>(this);
        }
    }
    //! DEBUGGING ONLY: Get current strong ref count.
    inline int32_t getStrongCount() const {
        return mCount.load(std::memory_order_relaxed);
    }

    typedef LightRefBase<T> basetype;

protected:
    // 注意，析构函数是 protected，这就限定了 LightRefBase 只能被继承
    inline ~LightRefBase() { }

private:
    mutable std::atomic<int32_t> mCount;
};
```

`incStrong` 的实现很简单，由于引用计数 `mCount` 不跟其他数据具有依赖关系，这里直接用 `memory_order_relaxed` 就可以了。

`decStrong` 里，`fetch_sub` 在给 `mCount` 减 1 的同时返回了原来的值，如果旧值是 1，说明我们是最后一个引用对象的人，接下来就改删除对象了。`if` 语句里的 `atomic_thread_fence` 和 `fetch_sub` 构成了一个 *Atomic-fence synchronization*：
```
class Foo : public LightRefBase { ... };

Thread A                                Thread B
---------                               -----------
do with Foo*                            do with Foo*
fetch_sub(1, memory_order_release)
  => mCount = 1
                                        fetch_sub(1, memory_order_release)
                                        atomic_thread_fense(memory_order_acquire)
                                        delete Foo*
```
假定存在这么一个对象 Foo*，它同时被线程 A、B 引用。线程 A 使用完以后，先执行了 `decStrong`；在线程 B `decStrong` 的时候，检查到 `mCount` 的旧值为 1，于是执行 `if` 语句中的内容。

所谓的 *Atomic-fense synchronization* 就是，在线程 A 中，do with Foo* 比 fetch_sub 先执行；线程 A 的 fetch_sub 比 线程 B 的 fetch_sub 先执行；线程 B 的 atomic_thread_fense 保证了 fetch_sub 比 delete Foo* 先执行。所以，线程 B 删除对象的引用的时候，线程 A 的 do with Foo* 一定以及执行完了。如果没有这个 fense，那么 delete Foo* 就可以在 fetch_sub 前面执行，而此时可能其他线程还在使用该对象。

原因 momory order 和 atomic_thread_fense 的更多信息，读者可以参考 [memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) 和 [atomic_thread_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)。

说了这么多，你可能就会想问，有没有其他更简单的方法来实现 `decStrong` 呢？有的
```C++
void decStrong(__attribute__((unused)) const void* id) const {
    if (mCount.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        delete static_cast<const T*>(this);
    }
}
```
问题在于，`if` 语句的条件只在最后一个持有引用的人执行时才会为 `true`；这种情况下，我们才需要一个 `acquire operation`。这是通过性能换取代码复杂性的一个例子。

## 智能指针的实现

智能指针的实现是 `sp`，这里我们只看最关键的几个函数：
```C++
// system/core/libutils/include/utils/StrongPointer.h
template<typename T>
class sp {
public:
    sp(T* other);  // NOLINT(implicit)
    ~sp();

    inline T&       operator* () const     { return *m_ptr; }
    inline T*       operator-> () const    { return m_ptr;  }
    inline T*       get() const            { return m_ptr; }
    inline explicit operator bool () const { return m_ptr != nullptr; }

private:    
    T* m_ptr;
};
```

直接通过指针来构造 sp 的情况很简单，如果传入的指针非空，就调用 `incStrong` 增加一个引用计数
```C++
template<typename T>
sp<T>::sp(T* other)
        : m_ptr(other) {
    if (other)
        other->incStrong(this);
}
```

析构的时候，调用 `decStrong`，如果引用计数降为 0，`decStrong` 将会删除 `m_ptr`。
```C++
template<typename T>
sp<T>::~sp() {
    if (m_ptr)
        m_ptr->decStrong(this);
}
```

## LightRefBase 设计问题

不得不说，轻量级指针除了 decStrong 的原子操作比较费解外，其他实现都是非常直观的。但如果我们留心观察，还是能够找到一些闪光点的。

### 为什么要让被管理对象继承 LightRefBase

从易用性的角度考虑，如果被管理对象（如前面例子里的 Foo）不需要继承 `LightRefBase`，无疑用起来会更加的方便。在考虑这个问题的时候，不妨看一看 `LightRefBase` 都有什么成员变量。可以看到，我们把引用计数存放在了 `LightRefBase` 里。这样一来，在堆上创建对象的时候，我们只需要分配一次内存。

如果不这样做，我们将不得不在堆上分配多一个对象，用来保存引用的计数：
```C++
#include <atomic>
#include <iostream>

template<typename T>
class NotSoLightSP {
 public:
  NotSoLightSP(T* ptr)
      : count_{new std::atomic<int>{1}}, ptr_{ptr} {}
  NotSoLightSP(const NotSoLightSP<T>& other)
      : count_{other.count_}, ptr_{other.ptr_} {
    count_->fetch_add(1, std::memory_order_relaxed);
  }

  ~NotSoLightSP() {
    if (count_->fetch_sub(1, std::memory_order_release) == 1) {
      std::atomic_thread_fence(std::memory_order_acquire);
      delete count_;
      delete ptr_;
    }
  }

  int RefCount() const { return count_->load(std::memory_order_relaxed); }

 private:
  std::atomic<int>* count_;
  T* ptr_;
};

class Foo {
 public:
  Foo() {
    std::cout << "Foo()" << std::endl;
  }
  ~Foo() {
    std::cout << "~Foo()" << std::endl;
  }
};

int main() {
  NotSoLightSP<Foo> sp{new Foo};
  {
    auto sp2 = sp;
    std::cout << "ref count = " << sp2.RefCount() << std::endl;
  }
  std::cout << "ref count = " << sp.RefCount() << std::endl;
}
```
输出为：
```
Foo()
ref count = 2
ref count = 1
~Foo()
```

最后，我们再参考 C++ 标准库的 `shared_ptr` 来讲一讲。对 `shared_ptr` 来说，一般情况下它也是需要在堆上分配对一个对象用于保存引用计数的；不那么一般的情况是，`make_shared`，这个时候它可以分配一个大的内存块，然后使用 placement new 来构造这两个对象（待管理对象和引用计数都放在这个内存块中）。

### 为什么 LightRefBase 是一个类模板（而 RefBase 却不是）

如果读者还没了解过 `RefBase`，可以在我发完下一篇文章的时候再回来看这一小节。

细心的读者应该留意到，我们是这样继承 LightRefBase 的：
```C++
class Foo : public LightRefBase<Foo> { ... }
```
而 `RefBase` 是这样的：
```C++
class Bar : public RefBase { ... }
```
要寻找这个问题的答案，可以看看 `LightRefBase` 用模板参数来做了什么：
```C++
inline void decStrong(__attribute__((unused)) const void* id) const {
    if (mCount.fetch_sub(1, std::memory_order_release) == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        delete static_cast<const T*>(this);
    }
}
```
可以看到，整个类唯一使用了 `T` 的地方就是这里。由于 `LightRefBase` 没有虚函数，所以在 delete this 指针的时候，需要把它强制转换为真正的类型 `T*`，否则将不会执行 `T::~T()`。另一方面，`RefBase::~Refbase()` 是一个虚函数，所以 `RefBase` 不需要把 this 强转回真正的类型就能够 delete this。

`LightRefBase` 之所以搞得这么麻烦，和前一个问题一样，都是为了性能。如果把析构函数是虚函数，那么每个子类都将多消耗一个指针用于存储函数表，这样就不够“light”(轻量)了。



