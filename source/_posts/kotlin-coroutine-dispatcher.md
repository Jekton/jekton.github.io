---
title: Kotlin 协程到底运行在哪个线程里
date: 2020-03-31 15:24:39
categories: Kotlin
tags: [Kotlin, Coroutine]
description: 与其说协程是一个轻量级线程，我更愿意把它当然一个个待执行/可执行的任务。这样就引申出一个问题——协程是运行在哪个线程上的？这就是本篇文章想要探讨的问题，同时我们也将学习如何让协程在特定的线程里执行。
---

与其说协程是一个轻量级线程，我更愿意把它当然一个个待执行/可执行的任务。这样就引申出一个问题——协程是运行在哪个线程上的？这就是本篇文章想要探讨的问题，同时我们也将学习如何让协程在特定的线程里执行。

首先来看一个例子：
```kotlin
fun log(msg: String) {
    println("[${Thread.currentThread().name}] $msg")
}

fun main() = runBlocking {
    val job1 = GlobalScope.launch {
        log("launch before delay")
        delay(100)
        log("launch after delay")
    }
    val job2 = GlobalScope.launch {
        log("launch2 before delay")
        delay(200)
        log("launch2 after delay")
    }

    job1.join()
    job2.join()
}
```
下面是在我机器上的一个输出（需要加入 JVM 参数 `-Dkotlinx.coroutines.debug` 才会打印协程名）：
```
[DefaultDispatcher-worker-2 @coroutine#3] launch2 before delay
[DefaultDispatcher-worker-1 @coroutine#2] launch before delay
[DefaultDispatcher-worker-1 @coroutine#2] launch after delay
[DefaultDispatcher-worker-1 @coroutine#3] launch2 after delay
```
这个输出有两个要点：
1. 从线程名推断，两个协程很可能运行在某个线程池中
2. 第二个协程先运行在 worker-2，然后又运行在 worker-1

关于第一点， `launch` 的文档有这么一句话：
> If the context does not have any dispatcher nor any other `ContinuationInterceptor`, then `Dispatchers.Default` is used.

这里引入了本篇文章的主题——Distpacher，正是它决定了协程运行在哪个线程里。Dispatcher 的问题我们马上会谈到，我们先看看第二个问题。

第二个协程先运行在 worker-2，然后又运行在 worker-1。这提醒我们，很多时候不能假设协程会运行在同一个线程里，它唯一保证的是，协程中的代码会串行执行。由于协程是串行执行的，即使前后不是在同一个线程，我们也能安全地对**局部变量**进行读写：
```kotlin
suspend fun foo() {
	val list = someSuspendFn()
	val list2 = someOtherSuspendFn()
	// cope with list/list2
}
```

现在我们回到本篇文章的重点——Dispatcher。所谓的 Dispatcher，中文我们可以叫它分发器，是用来将协程分发到特定线程去执行的。它的接口是 `ContinuationInterceptor`：
```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
	fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
}
```
注意，它继承了 `CoroutineContext.Element`，所以他也是一个 context。

`Continuation` 代表着协程运行的某个中间运行状态：
```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```
当需要继续执行协程时，Kotlin 会调用它的 `resumeWith` 方法。`ContinuationInterceptor` 的 `interceptContinuation` 可以返回一个新的 `Continuation`，在这个新的 `Continuation` 的 `resumeWith` 里面，我们可以让协程运行在任意的线程里。

下面我们看一个最简单的 Dispatcher：
```kotlin
class DummyDispatcher
    : AbstractCoroutineContextElement(ContinuationInterceptor),
      ContinuationInterceptor {

    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> {
        return continuation
    }
}

fun main() = runBlocking {
    val dispatcher = DummyDispatcher()
    log("which thread am I in?")
    val job = GlobalScope.launch(dispatcher) {
        log("launch before delay")
        delay(100)
        log("launch after delay")
    }

    job.join()
}
```
由于我们的 `DummyDispatcher` 什么也没做，协程会继续在原来的线程中执行。运行结果为：
```
[main @coroutine#1] which thread am I in?
[main @coroutine#1] launch before delay
[kotlinx.coroutines.DefaultExecutor] launch after delay
```
一开始 `runBlocking` 所在的线程为 main，由于我们返回了原始的 `continuation`，所以在 `delay` 前的那个调用还是在线程 main；最后一个打印之所以换到了另一个线程，是因为 `delay` 内部使用了这个 DefaultExecutor 来实现延迟执行。

现在我们来构造一个真正的 `ContinuationInterceptor`，它会让协程运行在我们创建的线程池里：
```kotlin
class SingleThreadedDispatcher
    : AbstractCoroutineContextElement(ContinuationInterceptor),
      ContinuationInterceptor {

    private val executorService = Executors.newSingleThreadExecutor {
        Thread(it, "SingleThreadedDispatcher")
    }

    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> {
        return OurContinuation(continuation)
    }

    inner class OurContinuation<T>(private val origin: Continuation<T>) : Continuation<T> {
        override val context: CoroutineContext = origin.context
        override fun resumeWith(result: Result<T>) {
            executorService.submit {
            	// origin.resumeWith 会执行协程中的代码。在这里调用的话，协程就
            	// 能运行在我们自定义的线程池中了
                origin.resumeWith(result)
            }
        }
    }

    fun close() {
        executorService.shutdown()
    }
}

fun main() = runBlocking {
    val dispatcher = SingleThreadedDispatcher()
    log("which thread am I in?")
    val job = GlobalScope.launch(dispatcher) {
        log("launch before delay")
        delay(100)
        log("launch after delay")
    }

    job.join()
    dispatcher.close()
}
```
程序的输出为：
```
[main @coroutine#1] which thread am I in?
[SingleThreadedDispatcher] launch before delay
[SingleThreadedDispatcher] launch after delay
```
可以看到，它确实运行在我们定义的那个线程池中。

注意到每次调用 `executorService.submit` 我们都创建了一个新对象，还可以复用 `OurContinuation`：
```kotlin
class SingleThreadedDispatcher {
	// ...

    inner class OurContinuation<T>(private val origin: Continuation<T>)
        : Continuation<T>, Runnable {
        override val context: CoroutineContext = origin.context

        private var result: Result<T>? = null
        override fun resumeWith(result: Result<T>) {
            this.result = result
            executorService.submit(this)
        }
        override fun run() {
            origin.resumeWith(result as Result<T>)
        }
    }
}

```

眼尖的你有没有留意到，虽然我一直在说 Dispatcher，可是前面我们实现的却是 `ContinuationInterceptor`，是 **interceptor**啊！其实正常情况下，我们一般不会直接来实现 `ContinuationInterceptor`，而是实现它的子类 `CoroutineDispatcher`，这就是为什么我们都说 Dispatcher 而不是 Interceptor。本质上他们的实现和前面的 `SingleThreadedDispatcher` 是一样的，读者可以自行阅读源码了解一下。

最后一个问题是，在哪里调用的 `interceptContinuation`？答案可以在 `suspendCoroutine` 的实现里找到：
```kotlin
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T =
    suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
```
通过 coroutine.intrisics 创建的协程都是 unintercepted 的，一般情况下，我们会调用 `intercepted()` 方法对它做一个拦截：
```kotlin
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this

internal abstract class ContinuationImpl {
	// ...

	public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
}
```
编译器会将 `suspend` 函数转成一个 `Continuation`，这个 `Continuation` 就是 `ContinuationImpl`；所以最终执行 `interceptContinuation`，如果 context 里有 `ContinuationInterceptor` 的话。