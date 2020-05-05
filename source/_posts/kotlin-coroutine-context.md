---
title: kotlin 协程上下文那点事
date: 2020-03-28 14:42:04
categories: Kotlin
tags: [Kotlin, Coroutine]
description: 用线程做类比的话，协程的 context 可以认为是协程的“线程私有变量”，同时这个私有变量是不可变的。也就是说，我们在创建一个协程的时候，他的 context 携带的信息就已经确定了下来
---



用线程做类比的话，协程的 `context` 可以认为是协程的“线程私有变量”，同时这个私有变量是不可变的。也就是说，我们在创建一个协程的时候，他的 `context` 携带的信息就已经确定了下来。由于协程是很轻量的，在需要改变协程上下文的时候，我们可以直接新创建一个协程，所以不可变性不算是一个大问题。

它的两个跟我们我们日常开发联系比较大的方法如下：
```kotlin
@SinceKotlin("1.3")
public interface CoroutineContext {
	interface Key<E : Element>
	
	public operator fun <E : Element> get(key: Key<E>): E?
	public operator fun plus(context: CoroutineContext): CoroutineContext
	
	// 注：省略了 fold 和 minusKey
}

```
`context` 可以认为是一个 `map`，通过它的 `get` 方法可以取出集合里的内容，比如：
```kotlin
val interceptor: ContinuationInterceptor? = context[ContinuationInterceptor]
```
`interceptor` 我们下篇文章再聊。这里用来做索引的 `ContinuationInterceptor` 其实是类 `ContinuationInterceptor` 的 companion object，并且这个伴生对象继承了 `CoroutineContext.Key<ContinuationInterceptor>`。初看起来这种写法有点奇怪，但习惯了以后还是不得不承认这个是很优雅的设计（相当于一个协变类型的 map）。下面我们看看如何给 `context` 带上自定义的信息。

`Element` 是集合里的内容：
```kotlin
interface CoroutineContext {
	// ...

    /**
     * An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
     */
    public interface Element : CoroutineContext {
        /**
         * A key of this coroutine context element.
         */
        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null
    }
}
```
实现它很简单：
```kotlin
class CoroutineName(val name: String) : CoroutineContext.Element {
	// companion object 是一个类对象，理所应当的，它能继承其他类
    companion object : CoroutineContext.Key<CoroutineName>
    override val key = CoroutineName
}

fun main() = runBlocking {
    GlobalScope.launch(Dispatchers.IO + CoroutineName("my coroutine")) {
    	// CoroutineName 的 companion object 继承的是 Key<CoroutineName>，
    	// 所以 coroutineContext[CoroutineName] 的类型是 CoroutineName？
        val name = coroutineContext[CoroutineName]?.name ?: "no name"
        println("name = $name")
    }
    Unit
}
// 打印：
// name = my coroutine

// 或者更常见的：
// 这里的 + 使用的是 CoroutineContext 里定义的 operator plus
someCoroutineScope.launch(Dispatchers.IO + CoroutineName("my coroutine")) {
    val name = coroutineContext[CoroutineName]?.name ?: return
    // ...
}
```
由于这种定义 `context` 是固定的模型，kotlin 还提供了一个便捷类：
```kotlin
class CoroutineName(val name: String) : AbstractCoroutineContextElement(CoroutineName) {
    companion object : CoroutineContext.Key<CoroutineName>
}
```

关于协程上下文，最后还剩下的问题是，我们如何拿到它。其中一种方法在上面的例子中我们已经看到：
```kotlin
public interface CoroutineScope {
    /**
     * The context of this scope.
     */
    public val coroutineContext: CoroutineContext
}
```
传递给 `CoroutineScope.launch()` 的 lambda 有一个类型为 `CoroutineScope` 的 receiver，这就是前面我们能够直接使用 `coroutineContext` 的原因。

另一个获取 `context` 的途径是 `Continuation`，例如：
```kotlin
interface Callback {
    fun onResult(enabled: Boolean)
}
fun isFeatureXEnabled(callback: Callback) {
    // ...
}

suspend fun isFeatureXEnabled() = suspendCoroutine { conn: Continuation<Boolean> ->
    isFeatureXEnabled(object : Callback {
        override fun onResult(enabled: Boolean) {
            val context = conn.context
            conn.resume(enabled)
        }
    })
}
```
`Continuation` 的 `context` 字段就是 `CoroutineContext`：
```kotlin
@SinceKotlin("1.3")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```
关于协程上下文的内容差不多就是这些了，如果读者有想分享出来的内容，欢迎给我留言。



