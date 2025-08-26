# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 共享可变状态与并发

协程可以使用像 `Dispatchers.Default` 这样的多线程调度器并行执行。这会带来所有常见的并行问题，主要问题是对共享可变状态的访问同步。协程领域中解决此问题的一些方案与多线程世界中的方案类似，但也有一些是独有的。

## 问题

让我们启动一百个协程，每个协程都执行相同的操作一千次。我们还将测量它们的完成时间以供后续比较：


```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // 要启动的协程数量
    val k = 1000 // 每个协程重复操作的次数
    val time = measureTimeMillis {
        coroutineScope { // 协程作用域
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}
```

我们首先用多线程 `Dispatchers.Default` 来递增一个共享可变变量，这是一个非常简单的操作。


```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        coroutineScope { // scope for coroutines
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}

var counter = 0

runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 10 ms
    Counter = 33248


## volatile 无济于事

有一个常见的误解是，将变量声明为 `volatile` 可以解决并发问题。让我们试一试：


```kotlin
@Volatile // 在 Kotlin 中，`volatile` 是一个注解
var counter = 0

runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 14 ms
    Counter = 27538


这段代码运行得更慢，但我们仍然不能总是在最后得到 "Counter = 100000"，因为 `volatile` 变量保证了对相应变量的线性化（这是“原子性”的技术术语）读写，但不提供更大操作（本例中的递增操作）的原子性。

## 线程安全的数据结构

适用于线程和协程的普遍解决方案是使用线程安全（也称为同步、线性化或原子）的数据结构，它为在共享状态上执行的相应操作提供了所有必要的同步。对于一个简单的计数器，我们可以使用 `AtomicInteger` 类，它具有原子性的 `incrementAndGet` 操作：


```kotlin
import java.util.concurrent.atomic.AtomicInteger

val counter = AtomicInteger()

runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 11 ms
    Counter = 100000


这是针对这个特定问题最快的解决方案。它适用于普通计数器、集合、队列以及其他标准数据结构及其基本操作。然而，它不容易扩展到复杂的状态或没有现成线程安全实现的复杂操作。

## 线程封闭：细粒度

**线程封闭** 是一种解决共享可变状态问题的途径，即对特定共享状态的所有访问都局限于单个线程。它通常用于 UI 应用程序，其中所有 UI 状态都局限于单个事件分发/应用程序线程。使用单线程上下文，可以很容易地将其应用于协程。


```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // 将每次递增限制在单线程上下文中
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 343 ms
    Counter = 100000


这段代码运行非常慢，因为它采用了_细粒度_的线程封闭。每次单独的递增都会使用 `withContext(counterContext)` 块从多线程的 `Dispatchers.Default` 上下文切换到单线程上下文。

## 线程封闭：粗粒度
实践中，线程封闭是以大块进行的，例如，大段的状态更新业务逻辑被限制在单个线程中。以下示例就是这样做的，从一开始就在单线程上下文中运行每个协程。


```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

runBlocking {
    // 将所有操作限制在单线程上下文中
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 2 ms
    Counter = 100000


## 互斥

互斥解决方案是通过_临界区_来保护对共享状态的所有修改，确保这些修改永不并发执行。在阻塞世界中，你通常会为此使用 `synchronized` 或 `ReentrantLock`。协程的替代方案称为 `Mutex`。它具有 `lock` 和 `unlock` 函数来划定临界区。关键区别在于 `Mutex.lock()` 是一个挂起函数。它不会阻塞线程。

还有一个 `withLock` 扩展函数，它方便地表示了 `mutex.lock(); try { ... } finally { mutex.unlock() }` 模式：


```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

val mutex = Mutex()
var counter = 0

runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // 用锁保护每次递增
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

    Completed 100000 actions in 153 ms
    Counter = 100000


这个例子中的加锁是细粒度的，因此会付出性能代价。然而，对于某些情况，这是一个不错的选择，在这些情况下，你必须定期修改某些共享状态，但没有一个自然线程来限定这个状态。
