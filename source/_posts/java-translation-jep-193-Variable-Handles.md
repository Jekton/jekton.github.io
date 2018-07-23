---
title: 翻译 - JEP 193：Variable Handles
date: 2018-07-22 13:10:24
categories: Java
tags: Java
description: 定义一个用来操作对象的字段、数组元素的跟 `"java.util.concurrent.atomic"` 和 `"sun.misc.Unsafe"` 等价的标准工具，它提供了一个标准的栅栏操作（fence operation）集用于精细地控制内存排序和一个标准的可达性栅栏操作（reachability-fence operation）来保证一个被引用的对象是强可达的（strongly reachable）。
---

Author: Doug Lea
Owner: Paul Sandoz
Type: Feature
Scope: SE
Status: Closed/Delivered
Release: 9
Component: core-libs/java.lang
Discussion: core dash libs dash dev at openjdk dot java dot net
Effort: M
Duration: L
Relates to [JEP 266: More Concurrency Updates](http://openjdk.java.net/jeps/266)
Reviewed by: Dave Dice, Paul Sandoz
Endorsed by: Brian Goetz
Created: 2014/01/06 20:00
Updated: 2017/08/17 16:45
Issue: [8046183](https://bugs.openjdk.java.net/browse/JDK-8046183)

> 原文地址：[JEP 193: Variable Handles](http://openjdk.java.net/jeps/193)，JEP 表示 JDK Enhancement Proposal

## 摘要

定义一个用来操作对象的字段、数组元素的跟 `java.util.concurrent.atomic` 和 `sun.misc.Unsafe` 等价的标准工具，它提供了一个标准的栅栏操作（fence operation）工具集用于精细地控制内存排序和一个标准的可达性栅栏操作（reachability-fence operation）来保证一个被引用的对象是强可达的（strongly reachable）。


## 目标

下面是一些必须达到的目标：
- 安全性。它不能使 JVM 处于一个不一致的状态。例如，对象的某个字段只能用一个能够转换成对应类型的（castable to the field type）值来更新，一个数组的元素只有当下标处于正常范围内才能够被访问。
- 一致性（Integrity）。访问对象的字段遵循跟 `getfield`、`putfield` 这两个指令一样的访问权限控制，并且 `final` 域不能被更新。（`MethodHandles` 在读写成员变量时同样遵守这里所说的安全和一致性规则）
- 性能。它所提供的性能必须跟 `sun.misc.Unsafe` 差不多（特别地，除了某些无法折叠的安全检查，生成的汇编代码应该几乎完全相同。
- 可用性。它的 API 必须必 `sun.misc.Unsafe` 更好。

我们希望它的 API 能够比 `java.util.concurrent.atomic` 更好，但这不是必须的。


## 动机

随着 Java 并发、并行编程的发展，程序员们对于不能对类的成员执行原子操作或对操作进行排序感到越来越失望；比方说，原子地递增一个用于计数的成员变量。到目前为止，唯一能够实现这一目标的方法就是使用一个独立的 `AtomicInteger`（不仅增加了内存，还因为这个额外的间接性引入了其他并发问题）；或者，在某些情况下，使用一个原子的 `FieldUpdaters` （通常带来更多的性能损耗（overhead））；亦或者，使用不安全的用于 JVM 的基础设施（JVM intrinsics）`sun.misc.Unsafe`（它经常是不可移植且不可用的）。

如果没有这个 JEP，随着对 Java 内存模型的修改，这些问题会因原子 API 扩充对访问一致性（access-consistency）（对应于 C++11 的内存模型）的支持而变得更糟糕。


## 描述

一个变量句柄（variable handle）是一个变量的带类型的引用，它支持使用一系列访问模型对变量进行读写。支持的变量类型包括成员变量、静态成员变量和数组元素。另一些正在考虑是否支持的类型有数组视图（array views），它把一个 `byte` 或 `char` 数组当成 `long` 数组；就像 `ByteBuffer` 用来描述一个堆外内存（off-heap regions）一样。

变量句柄需要我们增强标准库、JVM，增加编译器的支持。此外，它还需要对 Java 语言规范和 Java 虚拟机规范进行小的修改。一个小小的语言增强，用于编译时的类型检查并且补足现有的语法，也是需要考虑的。

规范应该通过一种自然的方式来扩展额外的类基本类型（primitive-like）数值类型和类数组（array-like）类型，如果它们曾经被添加到 Java 里。这不是一个通用的用来访问、更新多个变量的事务机制。其他可选的用来表述、实现这些构件的方案可能会在这篇 JEP 里探讨，也可能是在更进一步的 JEP 里。

变量句柄使用一个抽象类 `java.lang.invoke.VarHandle` 来表示，每个变量的访问模式用[多态签名（signature-polymorphic）](https://docs.oracle.com/javase/8/docs/api/java/lang/invoke/MethodHandle.html#sigpoly)来表示。

> 小罗路过：关于多态签名，在后面看了 `VarHandle` 的方法后读者就会明白的。

访问模式（access mode）代表一个最小可用集合，它被设计成跟 C/C++ 11 的原子变量相兼容而不是依赖一个修改过的 Java 内存模型。如果需要的话，也可以添加额外的访问模式。某些变量可能不支持特定的访问模式，如果在对应的 `VarHandle` 上执行这些操作，将会抛出 `UnsupportedOperationException` 异常。

访问模式可以归纳为以下几类：
1. 读模式，例如用带 `volatile` 内存排序效果（volatile memory ordering effects）的语义去读一个变量；
2. 写模式，比如使用 release memory ordering effects 去更新变量；
3. 原子更新模式，例如使用带 volatile memory ordering effects 的 compare-and-set 去更新变量；
4. 数值原子更新模式，例如使用带 plain memory order effects 的写和用于读的 acquire memory order effects 来执行 get-and-add；
5. 按位原子更新模式，例如使用 release memory order effects 的写和 plain memory order effects 的读来执行 get-and-bitwise-and。

最后三个通常也称为 read-modify-write 模式。

访问模式方法的签名多态特性让变量句柄可以仅使用一个抽象类而支持各种各样的类型。这可以防止类型的爆炸。更进一步，尽管访问模式方法签名里的参数定义为 `Object` 数组，签名多态的特性仍然可以防止对基本类型的自动装箱操作并且不会将参数打包成数组。这使它们有了可预测的行为从而在 HotSpot 翻译器的运行时和 C1/C2 编译器上有更好的性能。

> 小罗路过：访问模式方法原文为 access mode method，指的是 `VarHandle` 的成员方法。

用于生成 `VarHandle` 的方法跟生成 `MethodHandle` 实例的方法放在了同一个地方，它们生成相等或类似的变量类型。

用于成员变量和静态变量的 `VarHandle` 使用 `java.lang.invoke.MethodHandles.Lookup` 下的方法生成，它通过查找接收类的字段来实例化对象。举个例子，通过查找来生成接收类 `Foo` 的 `int` 型成员 `i` 的 `VarHandle` 可以像下面这样来做：
```Java
class Foo {
    int i;

    // ...
}

class Bar {
    static final VarHandle VH_FOO_FIELD_I;

    static {
        try {
            VH_FOO_FIELD_I = MethodHandles.lookup().
                in(Foo.class).
                findVarHandle(Foo.class, "i", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```
这个查找过程在生成并返回 `VarHandle` 前，会检查一系列的访问控制权限。对 `MethodHandle` 来说也一样，会看提供了读、写的 `MethodHandle` （参考 `MethodHandles.Lookup` 的 `find{,Static}{Getter,Setter}` 方法）的针对特定字段是否有对应的权限。

> 小罗路过：举个栗子。下面把 `Foo.i` 改成了 `private`，所以 `VH_FOO_FIELD_I` 可以成功生成，而 `Bar.MH_FOO_FIELD_I` lookup 的时候会抛异常。

```Java
class Foo {
    static final VarHandle VH_FOO_FIELD_I;

    static {
        try {
            VH_FOO_FIELD_I = MethodHandles.lookup().
                    in(Foo.class).
                    findVarHandle(Foo.class, "i", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    private int i;
}

class Bar {
    static final MethodHandle MH_FOO_FIELD_I;

    static {
        try {
            MH_FOO_FIELD_I = MethodHandles.lookup().
                    in(Foo.class).
                    findSetter(Foo.class, "i", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

在下面这些条件下，访问模式方法会抛出 `UnsupportedOperationException` 异常：
- 对一个 `final` 变量调用写访问模式方法
- 对引用类型或非数值类型（如 `boolean`）调用数值访问模式方法（`getAndAdd`，`addAndGet`）
- 对引用类型或 `float/double` 执行按位访问模式方法（后者以后可能会移除）

一个字段不需要声明为 `volatile` 也可以使用 `VarHandle` 来进行 volatile access。实际上，如果携带了 `volatile` 修饰符，它会被忽略掉。这个行为跟 `java.util.concurrent.atomic.Atomic{Int, Long, Reference}FieldUpdater` 是不一样的，使用后者时对应的字段需要声明为 `volatile`。当我们在某些时候需要 volatile 语义而其他时候不需要时，FilldUpdater 就显得过于受限了。

生成用于数组的 `VarHandle` 位于 `java.lang.invoke.MethodHandles` （参考 `MethodHandles` 的 `arrayElement{Getter, Setter}` 方法）。例如，用于 `int` 数组的 `VarHandle` 可以这样生成：
```Java
VarHandle intArrayHandle = MethodHandles.arrayElementVarHandle(int[].class);
```
在下列情况下，访问模式方法会抛出 `UnsupportedOperationException` 异常：
- 使用数值方法模式方法去修改引用类型或非数值类型（如 `boolean`）数组的元素
- 对引用类型或 `float/double` 执行按位访问模式方法（后者以后可能会移除）


所有的变量类型的基本类型（primitive types）和引用类型都是被支持的，只要它们的变量种类（variable kinds）是成员变量、静态变量或数组。其他变量种类可能会部分或全部支持。

> 小罗路过：这里的变量种类指的是局部变量，成员变量这一些

生成用于 array-view-based 的 `VarHandle` 的方法位于 `java.lang.invoke.MethodHandles`。举个例子，下面生成的 `VarHandle` 把一个 `byte` 数组看成一个非对其（unaligned）的 `long` 数组：
```Java
VarHandle longArrayViewHandle = MethodHandles.byteArrayViewVarHandle(
        long[].class, java.nio.ByteOrder.BIG_ENDIAN);
```

尽管同样的效果可以通过 `java.nio.ByteBuffer` 得到，但这种方式需要一个 `ByteBuffer` 实例用于包裹 `byte` 数组。由于这导致了脆弱的逃逸分析，它并不总是能够得到可接受的性能并且每次访问都需要通过一个 `ByteBuffer` 实例。在非对其访问的情况下，除了普通（plain）的方法模式方法，都会抛出 `IllegalStateException` 异常。对齐访问的情况下，取决于变量的类型，一些 volatile 访问模式是允许的。这些 `VarHandle` 可以用来向量化（vectorize）数组存取操作。

访问模式方法的参数的数量、参数的类型、返回值的类型取决于变量种类（viriable kind）、变量类型和访问模式的特性。`VarHandle` 的生成方法（我们前面提到的那些）会在文档里说明必要条件。例如，对前面我们所生成的 `VH_FOO_FIELD_I` 调用 `compareAndSet` 需要 3 个参数，一个 Foo 实例作为接收者，一个 `int` 作为 expected value 和另一个作为 actual value：
```Java
Foo f = ...
boolean r = VH_FOO_FIELD_I.compareAndSet(f, 0, 1);
```
相对的，`getAndSet` 只需要两个参数，一个 Foo 实例作为接收者，一个 `int` 用于设置值：
```Java
int o = (int) VH_FOO_FIELD_I.getAndSet(f, 2);
```

访问数组元素的时候需要一个额外的 `int` 型的参数，它位于接收者和其他参数之间（如果有的话），这个参数对应于需要操作的元素的下标。

为了可预测的行为和运行时性能，`VarHandle` 实例必须放在一个 `static final` 的字段里（就跟 `Atomic{Int, Long, Reference}FieldUpdater` 所要求的那样）。这可以保证在调用访问模式方法的时候会发生常量折叠，例如去掉方法签名的检查和/或参数的类型转换检查。

> 注：将来的 HotSpot 增强可能会支持没有使用 `static final` 持有的 `VarHandle` 和 `MethodHandle`。

一个 `MethodHandle` 可以使用 `VarHandle` 的访问模式方法通过 `MethodHandles.Lookup.findVirtual` 来生成。例如，下面给一个特定的变量类型和变量种类生成一个 `compareAndSet` 访问模式方法对应的 `MethodHandle`：
```Java
Foo f = ...
MethodHandle mhToVhCompareAndSet = MethodHandles.publicLookup().findVirtual(
        VarHandle.class,
        "compareAndSet",
        MethodType.methodType(boolean.class, Foo.class, int.class, int.class));
```

`MethodHandle` 可以用一个变量种类和类型都兼容的 `VarHandle` 实例作为第一个参数来调用：
```Java
boolean r = (boolean) mhToVhCompareAndSet.invokeExact(VH_FOO_FIELD_I, f, 0, 1);
```
或者，`mhToVhCompareAndSet` 可以绑定到一个 `VarHandle` 实例然后再调用：
```Java
MethodHandle mhToBoundVhCompareAndSet = mhToVhCompareAndSet
        .bindTo(VH_FOO_FIELD_I);
boolean r = (boolean) mhToBoundVhCompareAndSet.invokeExact(f, 0, 1);
```
像这样的使用 `findVirtual` 进行的 `MethodHandle` 查找会使用一个 asType 转换来调整参数然后再返回结果。这个行为跟使用 `MethodHandles.invoker` 的类比物 `MethodHandles.varHandleInvoker` 来生成 `MethodHandle` 是一样的：
```Java
MethodHandle mhToVhCompareAndSet = MethodHandles.varHandleExactInvoker(
        VarHandle.AccessMode.COMPARE_AND_SET,
        MethodType.methodType(boolean.class, Foo.class, int.class, int.class));

boolean r = (boolean) mhToVhCompareAndSet.invokeExact(VH_FOO_FIELD_I, f, 0, 1);
```
所以通过包装在一个类中， `VarHandle` 可以在（类型被擦除）或反射的情景下使用。比方说，用来替代 `java.util.concurrent.Atomic*FieldUpdater/Atomic*Array` 中对 `Unsafe` 的使用（尽管需要更进一步的工作，以保证这些 updater 对相应的字段用于足够的访问权限）。

对访问模式方法的调用的编译跟具有签名多态的 `MethodHandle.invokeExact` 和 `MethodHandle.invoke` 所遵守的规则是一样的。下面这些是对 Java 语言规范所附加的内容：
1. 生成对 `VarHandle` 的签名多态的访问模式方法的引用
2. 允许签名多态方法返回不是 `Object` 类型的值，这意味着返回值类型不再是多态的（并且也因为可以在调用的地方声明一个强制类型转换）。这可以让写访问模式方法放回 `void`，`compareAndSet` 返回 `boolean` 变得更容易。

如果对签名多态的方法的调用行为可以增强为自动识别返回值的类型会很好，但这不是必须的。

> 注：使用像方法引用那样的语法来生成 `VarHandle` 和 `MethodHandle`，比方说 `VarHandle VH_FOO_FIELD_I = Foo::i`，它所需要的语法和运行时支持是可取的，但不会在这篇 JEP 里讨论。

运行时对访问模式方法的调用跟使用 `MethodHandle.invokeExact` 和 `MethodHandle.invoke` 进行签名多态方法调用所遵循的规则是类似的。下面是对 Java 虚拟机规范所附加的要求：
1. 在 `VarHandle` 内引用签名多态的访问模式方法
2. 定义对签名多态的访问模式方法进行 `invokevirtual` 时的行为。预期这种行为会通过一个从访问模式方法的调用到对应的 `MethodHandle` 之间的使用相同参数的转换来定义（参考前面对 `MethodHandles.Lookup.findVirtual` 的使用）。

> 小罗路过：这里第1条说我们可以拿到一个 access mode method（这些方法属于 `VarHandle`） 的 reference，所以自然就可以对这个 reference invokevirtual。通过使用类似 `MethodHandles.Lookup.findVirtual` 的机制生成 `MethodHandle` 后，就能够真正执行方法调用了（在这个意义上，我们可以认为这个 `MethodHandle` 实例对应着 `VarHandle` 的某一个 access mode method）。之所以这个转换是必须的，前面我们提到，对 `VarHandle` 的调用都会被折叠掉，所以也就不会有真正的方法存在。

`VarHandle` 对于所支持的变量类型、种类能够具有可靠的效率以达到目标性能要求是非常重要的。利用签名多态的方法可以避免自动装箱和数组的打包。（Java）实现必须：
- 在包 `java.lang.invoke` 的内部，HotSpot 将类中的 `final` 字段认为是真正的 final，这使得 `VarHandle` 被 `static final` 域引用的时候可以进行常量折叠。
- 利用 JDK 内部的 `@Stable` 为那些仅改变一次的值进行常量折叠，利用 `@ForceInline` 来保证方法即使已经达到普通方法的 inline 上限也会被 inline
- 使用 `sun.misc.Unsafe` 实现底层增强的 volatile 访问

一些 HotSpot 固有的支持（intrinsics）是必须的，部分罗列如下：
- 对 `Class.cast` 的支持，它已经被添加了（参考[JDK-8054492](https://bugs.openjdk.java.net/browse/JDK-8054492)）。在虚拟机添加这个支持前，一个常量折叠的 `Class.cast` 还会遗留冗余的检查，这会导致不必要的性能损失。
- 当并发访问时，acquire-get 访问模式能够与 set-release 访问模式进行同步（参考 `sun.misc.Unsafe.putOrdered{Int, Long, Object}`）。
- 对数组范围检查[JDK-8042997](https://bugs.openjdk.java.net/browse/JDK-8042997)的原生支持。静态方法可以被添加到 `java.util.Arrays` 来做这个检查，它接受一个待调用的函数，然后在检查出错的情况下，返回一个异常或者一个出错消息，这个错误消息可以被用于包含在一个待抛出的异常中。像这样的原生支持可以使用无符号数进行更好地比较（毕竟，数组长度总是正的）并且更好地把范围检查提升到一个被展开（unrolled）了的循环的外面进行检查。

此外，HotSpot 里更近一步的范围检查已经在 [JDK-8073480](https://bugs.openjdk.java.net/browse/JDK-8073480) 实现了（[JDK-8003585](https://bugs.openjdk.java.net/browse/JDK-8003585) 则用于强力去除 fork/join 框架、`HashMap` 和 `ConcurrentHashMap` 里的范围检查）。

`VarHandle` 的实现必须保持对 `java.lang.invoke` 包里的其他类的最小依赖，以避免启动时间的增加和在静态初始化时产生循环依赖。比方说，如果 `VarHandle` 的某些实现使用 `ConcurrentHashMap`，而 `ConcurrentHashMap` 也被修改成使用了 `VarHandle`，此时必须保证没有引入循环依赖。另一个更微妙的循环是 `ThreadLocalRandom` 和他对 `AtomicInteger` 的使用。保证 HotSpot 的 C2 编译器编译时间不会因为对 `VarHandle` 的使用而过度增加也是很值得要的。

> 小罗路过：`VarHandle` 是一个抽象类，这里 “`VarHandle` 的实现应该指的是它的子类”


## 内存栅栏（Memory fences）

> 小罗路过：memory fences 跟所谓的 memory barrier 是同一个东西，memory barrier 现在大多翻译为内存屏障。

屏障操作（fenced operations）作为 `VarHandle` 的静态方法来定义，是一个最小可用的精细控制内存顺序工具集。

```Java
/**
 * Ensures that loads and stores before the fence will not be
 * reordered with loads and stores after the fence.
 *
 * @apiNote Ignoring the many semantic differences from C and
 * C++, this method has memory ordering effects compatible with
 * atomic_thread_fence(memory_order_seq_cst)
 */
public static void fullFence() {}

/**
 * Ensures that loads before the fence will not be reordered with
 * loads and stores after the fence.
 *
 * @apiNote Ignoring the many semantic differences from C and
 * C++, this method has memory ordering effects compatible with
 * atomic_thread_fence(memory_order_acquire)
 */
public static void acquireFence() {}

/**
 * Ensures that loads and stores before the fence will not be
 * reordered with stores after the fence.
 *
 * @apiNote Ignoring the many semantic differences from C and
 * C++, this method has memory ordering effects compatible with
 * atomic_thread_fence(memory_order_release)
 */
public static void releaseFence() {}

/**
 * Ensures that loads before the fence will not be reordered with
 * loads after the fence.
 */
public static void loadLoadFence() {}

/**
 * Ensures that stores before the fence will not be reordered with
 * stores after the fence.
 */
public static void storeStoreFence() {}
```

一个 full fense 比 acquire fence 要更强一些（在对排序的保证这一意义上），后者又比 load load fence 更强。类似的，full fence 比 release fence 更强，后者比 store store fence 又更强。


## 可访问性栅栏（Reachability fence）

可访问行栅栏作为静态方法定义在 `java.lang.ref.Reference` 中：
```Java
class java.lang.ref.Reference {
   // add:

   /**
    * Ensures that the object referenced by the given reference
    * remains <em>strongly reachable</em> (as defined in the {@link
    * java.lang.ref} package documentation), regardless of any prior
    * actions of the program that might otherwise cause the object to
    * become unreachable; thus, the referenced object is not
    * reclaimable by garbage collection at least until after the
    * invocation of this method. Invocation of this method does not
    * itself initiate garbage collection or finalization.
    *
    * @param ref the reference. If null, this method has no effect.
    */
   public static void reachabilityFence(Object ref) {}

}
```
参考 [JDK-8133348](https://bugs.openjdk.java.net/browse/JDK-8133348)。

现在已经太迟了，无法添加一个类似于 `@Finalized` 的东西，用于修饰一个方法，使得在编译时或运行时对应的方法体看起来像下面这样：
```Java
    try {
        <method body>
    } finally {
        Reference.reachabilityFence(this);
    }
```

可以预感，类似的机制将会在某些编译期处理器得到支持。


## 其他选择

引入一种新形式的值类型（value type）用于支持 volatile 操作。然而，这会导致跟其他类型的性质不一致，程序员也需要付出更多努力来学习使用它。也考虑过依靠 `java.util.concurrent.atomic FieldUpdater`s 来完成这一目标，但它们的动态损耗（dynamic overhead）和使用限制使得这一选项并不适用的。

一些其他的选择，包括那些基于字段引用（field references）的方法在这些年都有人提出并讨论过，但最终因为语法上不可行、效率或者可用性问题消失了。

语法增强在这个 JEP 之前的版本考虑过，但被认为太过于奇异（magical）了。它重载了 `volatile` 关键字的语义并扩展到飘浮接口（译者注：with the overloaded use of the volatile keyword scoping to floating interfaces），一个用于引用类型而另一个用于所有支持的基本类型（primitive type）。

上一个版本的 JEP 也考虑过从 `VarHandle` 扩展出泛型类型（generic type），但这个带有多态签名的泛型加上对自动装箱类型的特殊对待，被认为是不成熟的。因为将来的 Java 版本会带有值类型（value type）、允许基于基本数据类型的泛型（参考[JEP-218](http://openjdk.java.net/jeps/218)）和一个增强的数组 [Arrays 2.0](http://cr.openjdk.java.net/~jrose/pres/201207-Arrays-2.pdf)。

基于特定实现的 `invokedynamic` 这一方法在这个 JEP 的之前版本也考虑过。这需要仔细地让带或不带 `invokedynamic` 的编译后的方法调用在语义上保持一致。此外，一些使用了 `invokedynamic` 的核心类，如 `ConcurrentHashMap` 将会导致循环依赖。


## 测试

压力测试将会使用 [jcstress](http://openjdk.java.net/projects/code-tools/jcstress/) 工具来开发。


## 风险和假设

有个 `VarHandle` 的原型实现已经使用 nano-benchmarks 和 fork/join benchmarks 进行了性能测试，其中 fork/join 使用了 `sun.misc.Unsafe` 的地方都替换成了 `VarHandle`。目前为止还没有发现明显的性能损失，HotSpot 上的问题也都不太麻烦（折叠掉强制类型转换检查和改进数组范围检查）。我们对这个方法的可行性是有信心的。尽管如此，我们也启动能够进行更多的实验，来保证在性能要求非常严格的环境下有可靠的编译技术，因为这种情况会更需要 `VarHandle`。


## 依赖

那些在包 `java.util.concurrent` 里的类（包括 JDK 中其他一下地方）会从 `sun.misc.Unsafe` 迁移到 `VarHandle`。

这篇 JEP 不依赖于 [JEP 188: Java Memory Model Update](http://openjdk.java.net/jeps/188)。



