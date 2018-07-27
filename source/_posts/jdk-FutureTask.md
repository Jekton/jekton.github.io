---
title: JDK 源码分析（1）- FutureTask
date: 2018-07-25 13:27:45
categories: Java
tags: JDK
---

本篇是 JDK 源码分析的第一篇，主要关注 `FutureTask` 的实现。我们所分析的源码使用的是 Java10，因此也假设了读者了解 `VarHandle`（Java9 新增的 API），不了解的读者可以看我翻译的[JEP 193](https://jekton.github.io/2018/07/22/java-translation-jep-193-Variable-Handles/)。

<!--more-->

## 用法简介

这里我们不准备详细讲 `FutureTask` 的用法，只是简单提一下，帮组读者回忆。

```Java
// 1. 首先我们构造一个 `FutureTask`
FutureTask<Foo> task = new FutureTask<>(callable);

// 2. 然后我们把这个实例放在后台线程执行，比方说 executor：
executor.execute(task);

// 3. 最后，我们获取执行的结果
Foo foo = task.get();
```

如果读者对 `FutureTask` 的用法不是很熟悉，等看完源码就会非常清楚了。


## 构造

```Java
public class FutureTask<V> implements RunnableFuture<V> {

    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;

    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        // this.state 是 volatile，对 volatile 字段的写入，存在一个 happen-before
        // 关系；也就是说，`this.state = NEW` 执行完毕时，`this.callable = callable`
        // 也保证已经写入
        this.state = NEW;       // ensure visibility of callable
    }
}

```

作为标准库的实现，在性能上我们锱铢必较。这里利用 `volatile` 的特性，可以不需要设置 `callable` 为 `volatile`。


## 任务执行

创建了 `FutureTask` 实例后，我们就可以执行他了。这个由 `run()` 方法来完成：
```Java
public class FutureTask<V> implements RunnableFuture<V> {

    public void run() {
        if (state != NEW ||
            // 这是一个原子操作
            !RUNNER.compareAndSet(this, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            // 假设有两个线程竞争执行这个 futureTask，线程1 执行了
            // state != NEW 后，轮到线程2，同样执行成功；接着线程2
            // 继续执行到方法结束并设置 runner 回 null；再后面，线
            // 程1 重新被调度，来到了下面这个 if 语句。此时其实已经
            // 执行完成，所以这里要再判断一次
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    // 情况1：发生异常
                    setException(ex);
                }
                if (ran)
                    // 情况2：执行成功
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            // 情况3：任务被取消
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }


    // VarHandle mechanics
    private static final VarHandle STATE;
    private static final VarHandle RUNNER;
    private static final VarHandle WAITERS;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            STATE = l.findVarHandle(FutureTask.class, "state", int.class);
            RUNNER = l.findVarHandle(FutureTask.class, "runner", Thread.class);
            WAITERS = l.findVarHandle(FutureTask.class, "waiters", WaitNode.class);
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }

        // Reduce the risk of rare disastrous classloading in first call to
        // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
        Class<?> ensureLoaded = LockSupport.class;
    }
}

```

接下来我们分 3 种情况来看代码。

### 情况1：发生异常

如果执行的过程发生了异常，会调用 `setException`：
```Java
public class FutureTask<V> implements RunnableFuture<V> {

    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes

    protected void setException(Throwable t) {
        // 可能客户会调用 cancel 方法取消任务，所以这里要用原子操作
        if (STATE.compareAndSet(this, NEW, COMPLETING)) {
            outcome = t;
            STATE.setRelease(this, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

}
```
`VarHandle` 的 `setRelease` 有这样一句注释：
> Sets the value of a variable to the {@code newValue}, and ensures that prior loads and stores are not reordered after this access.

所以（如果 `compareAndSet` 执行成功），当我们把 `state` 设置为 `EXCEPTIONAL` 前，能够保证 `outcome = t` 已经执行完成。

`finishCompletion` 我们留到后面再看。


### 情况2：执行成功

执行成功时调用 `set` 方法，它的实现跟 `setException` 差不多：
```Java
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state
        finishCompletion();
    }
}
```


### 情况3：任务被取消

如果需要取消任务，可以调用 `FutureTask` 的 `cancel` 方法：
```Java
public class FutureTask<V> implements RunnableFuture<V> {

    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW && STATE.compareAndSet
              (this, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            // 如果当前状态不是 NEW 并且不能成功将其设置为 INTERRUPTING/CANCELLED
            // 表示任务已经执行完，所以 cancel 失败
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    // 如果 t == null，要么是这个任务还没开始执行，要么已经执行
                    // 到了 run 方法里的 finally 块
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    STATE.setRelease(this, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }

}

```
如果 `mayInterruptIfRunning == false`，那就直接尝试把状态设置为 `CANCELLED`；否则需要 interrupt 线程。下面是 `run` 方法最后的操作：

```Java
public class FutureTask<V> implements RunnableFuture<V> {

    public void run() {
        // ...
        try {
            // ...
        } final {
            // ...

            int s = state;
            // 对应 mayInterruptIfRunning == true 的情况
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        // 注意，第一个比较的是参数 s，s 可能是 `INTERRUPTED`
        if (s == INTERRUPTING)
            // state 是 volatile，保证我们这里每次都能够读到最新的值
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }
}
```

所有 3 种情况，最后都会调用 `finishCompletion` 方法，这部分我们下一节继续看。


## 获取结果

在 `finishCompletion` 的内部，会唤醒等待结果的线程。这里我们先不看 `finishCompletion`，而是看看 `get` 方法，这样会更容易理解一些。
```Java
public class FutureTask<V> implements RunnableFuture<V> {

    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            // awaitDone 会返回结束时的状态
            s = awaitDone(false, 0L);
        return report(s);
    }

    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            // timeout 后还没有结束
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
}
```

这部分的重点在于 `awaitDone` 方法：
```Java
public class FutureTask<V> implements RunnableFuture<V> {

    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }


    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;

    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion or at timeout
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        // The code below is very delicate, to achieve these goals:
        // - call nanoTime exactly once for each call to park
        // - if nanos <= 0L, return promptly without allocation or nanoTime
        // - if nanos == Long.MIN_VALUE, don't underflow
        // - if nanos == Long.MAX_VALUE, and nanoTime is non-monotonic
        //   and we suffer a spurious wakeup, we will do no worse than
        //   to park-spin for a while
        long startTime = 0L;    // Special value 0L means not yet parked
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            int s = state;
            // 任务完成或被取消
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING)
                // 到 COMPLETING 状态了的话，预期很快就会结束，所以 yield 一下
                // 就够了
                // We may have already promised (via isDone) that we are done
                // so never return empty-handed or throw InterruptedException
                Thread.yield();
            else if (Thread.interrupted()) {
                // removeWaiter 把节点从 waiters 列表里移除
                removeWaiter(q);
                throw new InterruptedException();
            }
            // 第一个循环，q == null
            else if (q == null) {
                // 调用 get 方法时传入时间为 0 或负值，可以轮询任务；
                // 默认版本的 get 传入的 timed == false，会无限等待
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode();
            }
            // queued 初始为 false
            // 第二个循环会执行下面这个语句，把 q 入队。
            // （上一步成功的情况下）第三个循环才会开始执行再往下的休眠操作
            // 注意：每个循环都会检查 state
            else if (!queued)
                // 这里我们先把队头 waiters 赋值给 q.next，赋值语句的返回值还是
                // waiters。执行这个语句的时候，我们期待 waiter 的当前值是 waiters
                // 并且将它设置为 q。
                // 因为可能有多个线程同时执行这个方法，这个语句还是有可能会执行
                // 失败的。如果失败，在接下来的循环里会重试
                // weakCompareAndSet 就姑且当做原子的 compareAndSet 吧
                queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);
            else if (timed) {
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    // 把线程投入休眠
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this);
        }
    }

}

```

下面我们继续看 `removeWaiter`：
```Java
/**
 * Tries to unlink a timed-out or interrupted wait node to avoid
 * accumulating garbage.  Internal nodes are simply unspliced
 * without CAS since it is harmless if they are traversed anyway
 * by releasers.  To avoid effects of unsplicing from already
 * removed nodes, the list is retraversed in case of an apparent
 * race.  This is slow when there are a lot of nodes, but we don't
 * expect lists to be long enough to outweigh higher-overhead
 * schemes.
 */
private void removeWaiter(WaitNode node) {
    if (node != null) {
        node.thread = null;
        // 为了能够重新开始整个循环，包裹了一层 for 循环
        retry:
        for (;;) {          // restart on removeWaiter race
            // 遍历 waiters
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                if (q.thread != null)
                    pred = q;
                // 下面两种情况对应 q.thread == null，也就是说，q 是待删除的节点
                else if (pred != null) {
                    // next 字段是 volatile，所以这里可以直接赋值。把 prev.next
                    // 指向 s （q.next）后就删除了 q。
                    pred.next = s;
                    if (pred.thread == null) // check for race
                        // 很倒霉的，prev 也要被删除（或已经被删除）
                        // 这里有三种可能：
                        // 1. prev.node 刚被设置为 null，但这个节点还在列表里，
                        //    prev.next = s 成功把 q 从列表里删除
                        // 2. 节点已经被删除
                        //    2.1 prev 是头节点（这将会执行下面那个else if 子句），
                        //        q 被设置为队头，q.next 仍旧指向 s；
                        //    2.2 prev 不是头结点，prev.prev.next 指向了 q；
                        //    这两种情况下，q 还留在列表里
                        // 所有这3种情况下，数据结构都是正常的。如果 q 还在列表里，
                        // 重新开始循环后总会删除它。如果已经被删除，遍历完列表后
                        // 也会退出。
                        // q 已经被删除的情况下，即使下个循环里我们把别人要删除的
                        // 节点给删除了也没关系。
                        continue retry;
                }
                // q 在队头，所要把 waiters 设置为 s
                else if (!WAITERS.compareAndSet(this, q, s))
                    continue retry;
            }
            break;
        }
    }
}
```
如果你觉得 `removeWaiter` 难以理解，建议多看几遍（我自己也是研究了很久），使用原子操作来实现并发访问是会带来比较大的复杂度的。

最后，我们来看个简单的 `finishCompletion` 方法：

```Java
    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            // 把 waiters 设置为 null 后相当于取出所有的 waiter
            if (WAITERS.weakCompareAndSet(this, q, null)) {
                // 遍历 waiters
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        // 唤醒线程
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }

```


## 总结

`FutureTask` 到这里我们就看完了。虽然代码量不多，但还是比预料中的复杂。除了上面代码里明确使用的 volatile 和原子操作，`FutureTask` 对 `state` 的定义也是很讲究的。通过增序定义 state 常量，在某些情况下我们可以直接通过比较常量值来判断状态是否处于某些状态集。比方说，是否被中断使用的是 `state >= INTERRUPTING`；如果不这么做，我们就需要 `state == INTERRUPTING || state == INTERRUPTED`，毫无疑问，前者会执行得更快。
