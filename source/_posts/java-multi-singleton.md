---
title: 如何使用 Java 和 double-check 实现支持多实例的单例
date: 2019-05-26 20:13:52
categories: Java
tags: parallel-programming
---

考虑这样一个需求，我们有两个业务 A 和 B，他们共同使用一个硬盘缓存 `DiskCache` 的实现。由于在单个业务内只需要一份缓存，这很容易让我们想到单例模式。在本篇文章中，我们从最简单的传统的单例模式的实现开始，一步步实现一个优雅高效的多实例的单例模式。

<!-- more -->

首先我们看最简单情况——使用 double-check 实现单例：
```Java
public class DiskCache {

    private static volatile DiskCache sInstance;

    private int mMaxSize;

    public static DiskCache getInstance() {
        if (sInstance == null) {
            synchronized (DiskCache.class) {
                if (sInstance == null) {
                    sInstance = new DiskCache();
                }
            }
        }
        return sInstance;
    }

    private DiskCache() {
        mMaxSize = 2 * 1024 * 1024; // 2M
    }
}
```
为了便于后面进行更深入的讨论，这里需要强调一下 `volatile` 的作用。虚拟机执行 `sInstance = new DiskCache()` 这一行代码时，做了下面几件事：
1. 为对象分配内存
2. 初始化对象。也就是给 `mMaxSize` 赋值，设置 Class 指针等
3. 把对象的引用写到变量 `sInstance` 上

Java 的语言规范要求，任何线程如果读取到针对 `volatile` 变量的写操作的结果，那么这个写操作前的任何操作，都发生在这个读操作之前（happens-before 关系）。在我们的例子里，任何线程如果读到 `sInstance` 的值不为 `null`，`DiskCache` 的构造函数一定已经执行完成。

如果没有 `volatile`，某个线程可能会拿到一个 `sInstance`，但 `sInstance.mMaxSize` 为 0 或者任意数值。


现在我们考虑多个实例的情况。为了支持多个实例，我们把 `sInstance` 的类型改成 `DiskCache[]` 并定义几个常量代表相关的业务：
```Java
public class DiskCache {

    public static final int CLIENT0 = 0;
    public static final int CLIENT1 = 1;
    public static final int CLIENT2 = 2;
    public static final int CLIENT_COUNT = 3;

    private static volatile DiskCache[] sInstances;

    private int mMaxSize;

    public static DiskCache getInstance(int client) {
        if (sInstances == null || sInstances[client] == null) {
            synchronized (DiskCache.class) {
                // instantiate instance...
            }
        }
        return sInstances[client];
    }

    private DiskCache() {
        mMaxSize = 2 * 1024 * 1024; // 2M
    }
}
```

这个时候我犯难了，我们需要保证 `synchronized` 块里面执行初始化操作后，`sInstances[client]` 对其他线程是可见的并且对应对象的构造函数已经执行完成。

一个比较天真的实现可能像下面这样：
```Java
public static DiskCache getInstance(int client) {
    if (sInstances == null || sInstances[client] == null) {
        synchronized (DiskCache.class) {
            if (sInstances == null) {
                sInstances = new DiskCache[CLIENT_COUNT];
            }
            sInstances[client] = new DiskCache();
        }
    }
    return sInstances[client];
}
```
但我必须告诉大家，虽然 `sInstances` 是一个 `volatile` 变量，但我们对数组的内容的写操作并不会有任何的同步效果。

由此我们可能会想，能不能给每个实例一个 `volatile` 变量？当然可以！
```Java
public class DiskCache {

    public static final int CLIENT0 = 0;
    public static final int CLIENT1 = 1;
    public static final int CLIENT2 = 2;

    private static volatile DiskCache sInstance0;
    private static volatile DiskCache sInstance1;
    private static volatile DiskCache sInstance2;

    public static DiskCache getInstance(int client) {
        switch (client) {
            case CLIENT0:
                if (sInstance0 == null) {
                    synchronized (DiskCache.class) {
                        if (sInstance0 == null) {
                            sInstance0 = new DiskCache();
                        }
                    }
                }
                return sInstance0;
            case CLIENT1:
                if (sInstance1 == null) {
                    synchronized (DiskCache.class) {
                        if (sInstance1 == null) {
                            sInstance1 = new DiskCache();
                        }
                    }
                }
                return sInstance2;
            case CLIENT2:
                if (sInstance2 == null) {
                    synchronized (DiskCache.class) {
                        if (sInstance2 == null) {
                            sInstance2 = new DiskCache();
                        }
                    }
                }
                return sInstance2;
            default:
                throw new IllegalArgumentException("Unknown client " + client);
        }
    }
}
```
我可以很负责任地说，这段代码是正确的，并且他的运行效率很不错，就是难看了些，扩展性也不好。这意味着，我们还得回到使用数组的那个方法去。

回想一下前面我们关于 `volatile` happens-before 关系的论述，结合那个失败的基于数组的实现，我在想，是否有一种方式，让我们在把一个新创建的对象放到数组中后，再来写某个 `volatile` 变量；同时，在进入 `synchronized` 块之前，我们通过检查这个变量，来判断数组中对应的实例是否已经初始化。另外，由于我们只能写一个变量，位掩码也自然而然浮了出来。结合这几个点子，我们可以按下面这种方式来实现多实例的单例：
```Java
public class DiskCache {

    public static final int CLIENT0 = 0;
    public static final int CLIENT1 = 1;
    public static final int CLIENT2 = 2;
    private static final int CLIENT_COUNT = 3;

    private static final DiskCache[] sInstances = new DiskCache[CLIENT_COUNT];
    private static volatile int sInstanceMask;

    public static DiskCache getInstance(int client) {
        int mask = 1 << client;
        if ((sInstanceMask & mask) == 0) {
            synchronized (DiskCache.class) {
                if ((sInstanceMask & mask) == 0) {
                    sInstances[client] = new DiskCache();
                    sInstanceMask |= mask;
                }
            }
        }
        return sInstances[client];
    }
}
```

在这个实现中，如果客户读到了对应的位为 `1`，那么 `DiskCache` 一定是已经初始化完成，并且已经写到了数组里。需要注意的是，这里最多只支持 32 个实例。


