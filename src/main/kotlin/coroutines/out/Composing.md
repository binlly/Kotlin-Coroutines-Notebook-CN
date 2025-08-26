# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 组合挂起函数

## 默认是顺序执行的

我们使用正常的顺序调用，因为协程中的代码，就像常规代码一样，默认是_顺序执行_的。以下示例通过测量执行这两个挂起函数所需的总时间来演示这一点：


```kotlin
import kotlin.system.measureTimeMillis

suspend fun doSomethingUsefulOne(): Int {
    delay(500L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(500L) // pretend we are doing something useful here, too
    return 29
}
```


```kotlin
runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}
```

    The answer is 42
    Completed in 1005 ms


## 使用 async 实现并发
如果 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 的调用之间没有依赖关系，并且我们希望通过_并发_执行两者来更快地得到结果，该怎么办？这就是 `async` 发挥作用的地方。

从概念上讲，`async` 就像 `launch` 一样。它启动一个独立的协程，这是一个与所有其他协程并发工作的轻量级线程。不同之处在于，`launch` 返回一个 `Job` 并且不携带任何结果值，而 `async` 返回一个 `Deferred` — 一个轻量级的非阻塞未来（`future`），它代表了一个稍后提供结果的承诺。你可以在一个 `deferred` 值上使用 `.await()` 来获取其最终结果，但 `Deferred` 也是一个 `Job`，因此如果需要，你可以取消它。





```kotlin
runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

    The answer is 42
    Completed in 506 ms


## 惰性启动的 async

可选地，通过将其 `start` 参数设置为 `CoroutineStart.LAZY`，`async` 可以实现惰性启动。在此模式下，它仅在 `await` 需要其结果时，或者当其 `Job` 的 `start` 函数被调用时，才启动协程。运行以下示例：


```kotlin
runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        // some computation
        one.start() // start the first one
        two.start() // start the second one
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

    The answer is 42
    Completed in 505 ms


## 使用 async 实现结构化并发

如果 `concurrentSum` 函数的代码内部出现问题并抛出异常，所有在其作用域内启动的协程都将被取消。
取消总是通过协程层次结构传播：



```kotlin
runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> {
        try {
            delay(Long.MAX_VALUE) // Emulates very long computation
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> {
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

    Second child throws an exception
    First child was cancelled
    Computation failed with ArithmeticException

