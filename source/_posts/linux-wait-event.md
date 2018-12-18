---
title: Java 程序员眼里的 Linux 内核 —— wait_event 源码分析
date: 2018-12-15 16:30:37
categories: Linux
tags: [Linux, Java, parallel-programming]
description: 看 Linux 的 wait_event 源码时，联想到我们平时经常用得比较多的 wait/notify、double-check 和 volatile，突然意识 wait_event 简简单单几行代码的背后，涉及的知识非常丰富。本篇文章我们就一起了来探索它背后的知识，然后尝试着和我们的日常开发关联起来。
---

看 Linux 的 `wait_event` 源码时，联想到我们平时经常用得比较多的 wait/notify、double-check 和 `volatile`，突然意识 `wait_event` 简简单单几行代码的背后，涉及的知识点其实非常丰富。本篇文章我们就一起了来探索它背后的知识，然后尝试着和我们的日常开发关联起来。


# wait_event

> 这里使用 Linux-2.6.24 版本的源码

## 背景

在某些情况下，我们会需要等待某个事件，在这个事件发生前，把进程投入睡眠。比方说，同步写 IO；在发出写磁盘命令后，进程要进入休眠，等等磁盘完成。为了支持这一类场景，Linux 引入了 wait queue；wait queue 从概念上跟我们应用层使用的 condition queue 是一样的。

## 实现

这里我们着重讲 `wait_event` 的实现，一些相关的知识读者可以参考《深入理解LINUX内核》。

下面我们开始看代码：
```C
// ${linux_source}/include/linux/wait.h

/**
 * wait_event - sleep until a condition gets true
 * @wq: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_UNINTERRUPTIBLE) until the
 * @condition evaluates to true. The @condition is checked each time
 * the waitqueue @wq is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 */
#define wait_event(wq, condition)       \
do {                                    \
    if (condition)                      \
        break;                          \
    __wait_event(wq, condition);        \
} while (0)
```
这里只是先检测一遍条件，然后直接又调用 `__wait_event`：
```C
// ${linux_source}/include/linux/wait.h
#define __wait_event(wq, condition)                             \
do {                                                            \
    DEFINE_WAIT(__wait);                                        \
                                                                \
    for (;;) {                                                  \
        prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);    \
        if (condition)                                          \
            break;                                              \
        // schedule 使用调度器调度另一个线程去执行。当前线程被重新      \
        // 调度时，schedule 函数才会返回                            \
        schedule();                                             \
    }                                                           \
    finish_wait(&wq, &__wait);                                  \
} while (0)
```
`DEFINE_WAIT` 宏用于定义局部变量 `__wait`：
```C
// ${linux_source}/include/linux/wait.h
#define DEFINE_WAIT(name)                                   \
    wait_queue_t name = {                                   \
        .private    = current,                              \
        .func       = autoremove_wake_function,             \
        .task_list  = LIST_HEAD_INIT((name).task_list),     \
    }
```
`prepare_to_wait` 和 `finish_wait` 源码如下：
```C
// ${linux_source}/kernel/wait.c
/*
 * Note: we use "set_current_state()" _after_ the wait-queue add,
 * because we need a memory barrier there on SMP, so that any
 * wake-function that tests for the wait-queue being active
 * will be guaranteed to see waitqueue addition _or_ subsequent
 * tests in this thread will see the wakeup having taken place.
 *
 * The spin_unlock() itself is semi-permeable and only protects
 * one way (it only protects stuff inside the critical region and
 * stops them from bleeding out - it would still allow subsequent
 * loads to move into the critical region).
 */
void fastcall
prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
    unsigned long flags;

    // 非独占等待（可以同时唤醒多个进程）
    wait->flags &= ~WQ_FLAG_EXCLUSIVE;
    // 加锁
    spin_lock_irqsave(&q->lock, flags);
    // wait 不存在于某个等待队列时，才把它加入 q
    // wait 是我们新定义的，list_empty 将会返回 true
    if (list_empty(&wait->task_list))
        __add_wait_queue(q, wait);
    /*
     * don't alter the task state if this is just going to
     * queue an async wait queue callback
     */
    // 根据 wait 的定义，is_sync_wait() 这里会返回 true
    if (is_sync_wait(wait))
        // 前面注释说使用 set_current_state() 作为屏障，对此不理解的读者可以暂时忽略，
        // 后面我们会举例说明相关的用法
        set_current_state(state);
    // 解锁
    spin_unlock_irqrestore(&q->lock, flags);
}

/*
 * Used to distinguish between sync and async io wait context:
 * sync i/o typically specifies a NULL wait queue entry or a wait
 * queue entry bound to a task (current task) to wake up.
 * aio specifies a wait queue entry with an async notification
 * callback routine, not associated with any task.
 */
#define is_sync_wait(wait)    (!(wait) || ((wait)->private))

void fastcall finish_wait(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;

    __set_current_state(TASK_RUNNING);
    /*
     * We can check for list emptiness outside the lock
     * IFF:
     *  - we use the "careful" check that verifies both
     *    the next and prev pointers, so that there cannot
     *    be any half-pending updates in progress on other
     *    CPU's that we haven't seen yet (and that might
     *    still change the stack area.
     * and
     *  - all other users take the lock (ie we can only
     *    have _one_ other CPU that looks at or modifies
     *    the list).
     */
    if (!list_empty_careful(&wait->task_list)) {
        spin_lock_irqsave(&q->lock, flags);
        list_del_init(&wait->task_list);
        spin_unlock_irqrestore(&q->lock, flags);
    }
}
```

概括来讲，`prepare_to_wait` 把自己加入等待队列，`finish_wait` 则把自己从队列里移除。但由于 `prepare_to_wait` 可能会被调用多次，如果判断 `wait` 已经处于某个队列中，则不会重复添加。


# 条件、条件队列和锁

对于像我一样平时使用 Java 比较多的读者，对下面这一段代码一定不会觉得陌生：
```Java
synchronized (this) {
    while (!condition) {
        wait();
    }
    // do your stuff
}
```
这里我们不禁要问，应用层的代码可以这样简洁，为什么内核的就不行？这里我们先来大概了解一下条件队列，然后再回答这个问题。

所谓的条件队列/等待队列，一般由 3 个成分组成：
1. 一个队列，用于存放等待条件/事件的线程。在应用层，一般我们叫他条件队列（condition queue），LINUX 内核叫他 wait queue
2. 一个锁，用于保护这个队列
3. 一个谓词（它的计算结果为 bool 值）用作条件，即前面示例代码的 `condition`。

Java 程序员们在这里需要特别注意的是，我说的**锁的作用是保护条件队列**。回顾我们常写的 Java 代码，一般这个锁也用来保护谓词，但这个不是必须的。Java 要求我们在调用 `wait` 的时候必须持有锁的原因之一是，`wait` 的内部会把当前线程加入条件队列；修改列表必须持有锁（另一个原因是，`wait` 的语义之一便是执行后会释放锁，如果都不持有，何来的释放呢）。

但在另一面，唤醒条件队列上的线程却不一定需要持有锁，虽然 Java 要求我们必须持有锁才能调用 `notify`。持有锁调用 `notify` 的好处在于，notify 后条件不会改变。同时，如果持有锁的话，这个操作里也可以把相关线程从条件队列里删除。不好的地方在于，调用 notify 的线程在执行唤醒操作的时候还持有锁，被唤醒线程这个时候如果被内核调度，他的获取锁的操作将失败（会导致该线程又进入睡眠状态）。这种实现方式性能上可能差一点，但代码更安全。

不要求调用 notify 时持有锁的一个例子是 pthread。这种方式的问题在于，在 notify 还没执行完的时候，条件可能就发生了变化。可能的实现是，只设置线程为可执行状态，等线程获得锁后自己把自己从队列里面移除。

了解了相关的数据结构后，不难猜想 Java 里 `wait` 的实现。考虑一种应用层 wait 的实现如下：
```C
void wait() {
    add_to_condition_queue();
    unlock();
    schedule();
}
```
把 `wait` 方法做一下内联（inlining）处理，可以得到：
```C
lock();
while (!condition) {
    add_to_condition_queue();
    unlock();
    schedule();
}
// do your stuff
```
对比一下内核的 `wait_event`：
```C
void our_wait_event() {
    if (condition) return;

    for (;;) {
        // 如果你喜欢，换成 condition_queue 也可以
        add_to_wait_queue_if_not_added_yet();
        if (condition)
            break;
        schedule();
    }
    remove_from_wait_queue();
}
```

可以看到，内核把代码写得更复杂的好处在于，它在切换进程前可以再检查一次条件，如果条件已经满足，就不需要执行 `schedule` 了。切换进程需要保存当前进程的上下文，同时会导致 TLB、Cache 等一系列缓存时效，因此内核总是尽量避免不必要的线程切换，而代价就是更复杂的代码。


# double-check

首先，如果你也和我一样觉得 `our_wait_event` 里面两个 `if` 有点难看，我们不妨试着来给他改一改：
```C
void our_wait_event2() {
    while (!condition) {
        add_to_wait_queue_if_not_added_yet();
        schedule();
    }
    remove_from_wait_queue();
}
```
咋一看好像没什么问题，都是一样的检测条件，在条件不满足的情况下加入等待队列，调用 `schedule`。重要的是，上面这段代码更简洁，更易读。那么，他正确吗？

不消说，肯定是有问题的，不然那班内核程序员不会不知道该这么写。那问题究竟出在哪里呢？

考虑下面两个执行流：
```
thread1                              thread2
-----------------------           --------------
check condition => false
add_to_wait_queue()

                                  alter_condition()
                                  notify_all()

state = TASK_UNINTERRUPTABLE
schedule()
```
thread1 在把自己加入等待队列后，schedule 前，thread2 就更改了条件并且调用 notify。在这种情况下，如果没有其他线程再次调用 notify，thread1 将会永远休眠（而 thread2 认为自己已经 noitfy 过 thread1 了）。

为了防止发生这种情况，在添加到等待队列后，thread1 还应该再检查一次条件，如果条件满足，直接把自己从队列里移除就可以了。

为了方便读者把 `wait_event` 和 double-check 联系起来，下面我们看一段使用 double-check 实现的 Java 的单例的例子：
```Java
public static SomeClass getInstance() {
    if (sInstance == null) {
        synchronized (SomeClass.class) {
            if (sInstance != null) {
                sInstance = new SomeClass();
            }
        }
    }
    return sInstance;
}
```

两者的共同点都是先检测一遍条件是否成立，然后设置一个“安全阀”。在持有这个安全阀时，再一次检测条件是否满足。double-check 的多线程安全性都源于这个安全阀。就 wait_event 来说，当我们把自己加入等待队列后，就可以保证不会丢失另一个线程的 notify。而创建单例时，加锁保证了第二次判断后不会有另一个线程同时创建对象。（可能说得有点抽象，如果读者不明白，直接跳过就好。只要读者能够完成下面的小测验，那就是懂得 double-check 的）。

## double-check 小测验

假设有这样一个方法，他可以用来下载文件：
```Java
interface DownloadCallback {
    void onSuccess(File file);
}

public void download(String url, DownloadCallback callback) {
    // ...
}
```
我们又假设，可能同时有多个客户会调用这个接口下载同一个文件。为了避免同时下载同一个文件，我们可以在下载时判断一下当前是否已经有任务在下载：
```Java

interface DownloadCallback {
    void onSuccess(File file);
}

private class DownloadTask {
    // guarded by itself
    private final List<DownloadCallback> mCallbacks = new ArrayList<>();

    public DownloadTask(String url, DownloadCallback callback) {
        mCallbacks.add(callback);
    }

    public void download() {
        // downloading ...
        File file = new File("downloaded-file");

        // bonus: 为什么需要拷贝 callback 列表？
        List<DownloadCallback> copy;
        synchronized (mCallbacks) {
            copy = new ArrayList<>(mCallbacks);
            mCallbacks.clear();
        }
        for (DownloadCallback callback : copy) {
            callback.onSuccess(file);
        }
    }

    public void addCallback(DownloadCallback callback) {
        synchronized (mCallbacks) {
            mCallbacks.add(callback);
        }
    }
}


private final ConcurrentMap<String, DownloadTask> mTasks = new ConcurrentHashMap<>();

public void download(String url, DownloadCallback callback) {
    File file = new File(getDestFilePath(url));
    if (file.exists()) {
        // 已经存在则不需要下载了
        callback.onSuccess(file);
        return;
    }
    DownloadTask task = new DownloadTask(url, callback);
    DownloadTask downloadingTask = mTasks.putIfAbsent(url, task);
    if (downloadingTask == null) {
        // 没有正在下载的任务时才需要下载
        task.download();
        return;
    }
    // 加入正在下载的任务的 callback 列表，
    downloadingTask.addCallback(callback);
}

private String getDestFilePath(String url) {
    return "url-to-file-path...";
}
```

为了检验读者是否真正理解 `wait_event`，你可以尝试着解决上面代码里存在的竞争条件。如果一时没能发现其中的问题，建议读者再从头读一遍文章。为了鼓励读者独立思考、与他人交流，这里我就顺势偷个懒不公布答案了。毕竟，在实际工作中，可不是总会有人告诉你你的代码写得是否正确。


# 内存屏障一瞥

所谓的内存屏障，主要分为 3 种：
1. read memory barrier（rmb），保证屏障前的读发生在屏障后的读操作之前
2. write memory barrier（wmb），保证屏障前的写发生在屏障后的写操作之前
3. full memory barrier（mb），保证屏障前的读写操作发生在屏障后的读写操作之前

前面 `prepare_to_wait` 有这么一段注释：
```C
/*
 * Note: we use "set_current_state()" _after_ the wait-queue add,
 * because we need a memory barrier there on SMP, so that any
 * wake-function that tests for the wait-queue being active
 * will be guaranteed to see waitqueue addition _or_ subsequent
 * tests in this thread will see the wakeup having taken place.
 *
 * The spin_unlock() itself is semi-permeable and only protects
 * one way (it only protects stuff inside the critical region and
 * stops them from bleeding out - it would still allow subsequent
 * loads to move into the critical region).
 */
```
这段注释一开始我也是看得云里雾里，直到我找到了他们解决一个内核 bug 的[邮件](http://lkml.iu.edu/hypermail/linux/kernel/0312.1/1497.html)（Google 大法好）。

这里面的 tests for the wait-queue being active 可以根据 `waitqueue_active` 来理解，其实指的就是等待队列不为空。spin-lock 虽然可以防止数据竞争，但如果别人在检查的时候不去获取锁呢？（`waitqueue_active` 就没有加锁）。当然，不加锁可以获得更好的性能。
j
```C
static inline int waitqueue_active(wait_queue_head_t *q)
{
    return !list_empty(&q->task_list);
}
```
`set_current_state` 使用 `set_mb` 来设置当前进程的状态。
```C
/*
 * set_current_state() includes a barrier so that the write of current->state
 * is correctly serialised wrt the caller's subsequent test of whether to
 * actually sleep:
 *
 *    set_current_state(TASK_UNINTERRUPTIBLE);
 *    if (do_i_need_to_sleep())
 *        schedule();
 *
 * If the caller does not need such serialisation then use __set_current_state()
 */
#define __set_current_state(state_value)                \
    do { current->state = (state_value); } while (0)
#define set_current_state(state_value)        \
    set_mb(current->state, (state_value))
```

下面是文档对 `set_mb` 的描述：
```
 (*) set_mb(var, value)

     This assigns the value to the variable and then inserts a full memory
     barrier after it, depending on the function.  It isn't guaranteed to
     insert anything more than a compiler barrier in a UP compilation.
```

`set_current_state` 在设置当前进程的状态后，会插入一个 mb。前面我们了解到，这将禁止 CPU 将 `set_current_state` 后面的 load/store 提前。

为了理解完全理解这里 `set_mb` 的使用，我们还需要再参考一下 `wake_up` 函数。但由于篇幅关系，这里我只是简单介绍它的实现：
```
for_each_process_in_wait_queue_without_lock:
    if process.state is sleeping:
        wake it up
        remove it from wait-queue
```

假设在 `prepare_to_wait` 里面我使用的是平凡的 `__set_current_state`，那么 CPU 就可以把 `prepare_to_wait` 函数返回后我们所执行的对条件的判断提前到设置进程状态前。这种情况下，如果发生以下的执行序列，CPU2 将会丢失一个 wake-up，他有可能会永远休眠。
```
CPU0                    CPU1
----------------        ------------------
                        check_condition() => false
condition = true
wake_up()
                        __set_current_state()
                        schedule()
```
解决办法就是引入一个 `mb`。在下面的例子中，如果 `__set_current_state()` 在 `wake_up()` 后执行，CPU1 上的这个线程将不会被唤醒，但随后的 `check_condition()` 会正确返回 `true`。反过来，如果 `__set_current_state()` 在 `wake_up()` 前执行，`check_condition()` 可能返回 `true` 也可能返回 `false`，但无论如何，他都不会丢失随后的 `wake_up()`。
```
CPU0                    CPU1
----------------        ------------------
condition = true
wake_up()
                        __set_current_state()
                        mb();
                        check_condition() => true
```
还记得吗，`set_current_state` 就是在设置进程状态后插入一个内存屏障，所以 `prepare_to_wait` 直接使用 `set_current_state` 就可以了。

现在，我们终于可以说自己完全理解 `wait_event` 了。也许读者是第一次接触内存屏障，但我敢保证，很多 Java 程序员在不知不觉中使用过一定形式上的屏障。下面我们看一个例子：
```Java
private int mSomeData;          // guarded by mDataSet
private int mSomeOtherData;     // guarded by mDataSet
private volatile boolean mDataSet;

public void foo() {
    if (mDataSet) {
        // you can now use mSomeData, mSomeOtherData
    }
}

public void bar() {
    mSomeData = 1;
    mSomeOtherData = 2;
    mDataSet = true;
}
```

有一定开发经验的读者很可能看过类似的代码。虽然我们没有在 `mSomeData` 和 `mSomeOtherData` 的读写上做显式的同步，但只要仔细编写代码，利用一个 `volatile` 变量 `mDataSet`，这段代码也可以是线程安全的。

为了避免引入内存屏障这个比较复杂的概念（并且提供更好的移植性），Java 使用一个 happens-before 来描述相关的概念。关于 `volatile` 有这么一条描述：

> A write to a `volatile` field happens-before every subsequent read of that field.

另外，对于同一个线程，有：
> If x and y are actions of the same thread and x comes before y in program order, then x happen before y.

同时，happens-before 具有传递性（x -> y, y -> z, x -> z），所以就有了下面这个结论：

对 `mSomeData, mSomeOtherData` 的写操作在 `mDataSet = true` 之前；`mDataSet = true` 在随后另一个线程对他的读操作之前；所以 `mSomeData, mSomeOtherData` 的写操作在随后对 `mDataSet` 的读操作之前。

直白一点说，只要一个线程看到 `mDataSet` 为 `true`，那他就一定能够正确读取到 `mSomeData, mSomeOtherData` 的值。

如果显式使用内存屏障，上面的代码就相当于：
```Java
private int mSomeData;
private int mSomeOtherData;
private boolean mDataSet;

public void foo() {
    if (mDataSet) {
        // 这个读内存屏障保证我们读到 mDataSet 后，也能读到 mSomeData/mSomeOtherData
        // 的最新值
        rmb();
        // you can now use mSomeData, mSomeOtherData
    }
}

public void bar() {
    mSomeData = 1;
    mSomeOtherData = 2;
    // 写内存屏障保证对 mSomeData/mSomeOtherData 的写在 mDataSet = true 之前执行
    wmb();
    mDataSet = true;
}
```

最后，墙裂推荐一本并行编程的神书《Is Parallel Programming Hard, And, If So, What Can You Do About It?》（可以免费获取），书里有关于内存屏障的最好的描述。



