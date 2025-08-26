# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 取消与超时

## 取消协程执行

在长时间运行的应用程序中，你可能需要对后台协程进行细粒度控制。 例如，用户可能关闭了启动协程的页面，现在其结果不再需要，并且其操作可以被取消。 `launch` 函数返回一个 `Job`，可用于取消正在运行的协程：


```kotlin
runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    main: Now I can quit.


一旦 `main` 调用 `job.cancel`，我们就不会看到来自另一个协程的任何输出，因为它已被取消。 还有一个 `Job` 扩展函数 `cancelAndJoin`， 它结合了 `cancel` 和 `join` 调用。

## 取消是协作性的

协程取消是_协作性的_。协程代码必须相互配合才能被取消。 `kotlinx.coroutines` 中的所有挂起函数都是_可取消的_。它们会检测协程的取消，并在取消时抛出 `CancellationException`。然而，如果一个协程正在进行计算并且不检测取消，那么它就无法被取消，如下例所示：


```kotlin
runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) { // computation loop, just wastes CPU
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    job: I'm sleeping 3 ...
    job: I'm sleeping 4 ...
    main: Now I can quit.


运行它会发现，即使在取消之后，它仍然会继续打印 "I'm sleeping"，直到该 `Job` 在五次迭代后自行完成。

通过捕获 `CancellationException` 但不重新抛出它，可以观察到同样的问题：


```kotlin
runBlocking {
    val job = launch(Dispatchers.Default) {
        repeat(5) { i ->
            try {
                // print a message twice a second
                println("job: I'm sleeping $i ...")
                delay(500)
            } catch (e: Exception) {
                // log the exception
                println(e)
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job="coroutine#70204":StandaloneCoroutine{Cancelling}@3db5defe
    job: I'm sleeping 3 ...
    kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job="coroutine#70204":StandaloneCoroutine{Cancelling}@3db5defe
    job: I'm sleeping 4 ...
    kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job="coroutine#70204":StandaloneCoroutine{Cancelling}@3db5defe
    main: Now I can quit.


虽然捕获 `Exception` 是一种反模式，但此问题可能会以更微妙的方式出现，例如在使用 `runCatching` 函数时，该函数不会重新抛出 `CancellationException`。

## 使计算代码可取消

有两种方法可以使计算代码可取消。 第一种是定期调用检查取消的挂起函数。 `yield` 和 `ensureActive` 函数是实现此目的的绝佳选择。 另一种方法是使用 `isActive` 显式检查取消状态。 让我们尝试后一种方法。

将上一个示例中的 `while (i < 5)` 替换为 `while (isActive)` 并重新运行。


```kotlin
runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // cancellable computation loop
            // prints a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    main: Now I can quit.


如你所见，现在这个循环被取消了。`isActive` 是一个扩展属性，通过 `CoroutineScope` 对象在协程内部可用。

## 使用 `finally` 关闭资源

可取消的挂起函数在取消时会抛出 `CancellationException`，可以按常规方式处理。 例如， `try {...} finally {...}` 表达式和 `Kotlin` 的 `use` 函数在协程取消时正常执行其终结操作：


```kotlin
runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("job: I'm running finally")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    job: I'm running finally
    main: Now I can quit.


`join` 和 `cancelAndJoin` 都会等待所有终结操作完成

## 运行不可取消的代码块

在上一示例的 `finally` 代码块中，任何尝试使用挂起函数的操作都会导致 `CancellationException`，因为运行此代码的协程已被取消。通常，这不是问题，因为所有行为良好的关闭操作（关闭文件、取消作业或关闭任何类型的通信通道）通常都是非阻塞的，并且不涉及任何挂起函数。但是，在极少数情况下，当你需要在已取消的协程中挂起时，你可以使用 `withContext` 函数和 `NonCancellable` 上下文，将相应的代码包装在 `withContext(NonCancellable) {...}` 中，如下例所示：


```kotlin
runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

    job: I'm sleeping 0 ...
    job: I'm sleeping 1 ...
    job: I'm sleeping 2 ...
    main: I'm tired of waiting!
    job: I'm running finally
    job: And I've just delayed for 1 sec because I'm non-cancellable
    main: Now I can quit.


## 超时

取消协程执行最明显的实际原因是其执行时间超过了某个超时。 虽然你可以手动跟踪对应 `Job` 的引用并启动一个单独的协程来在延迟后取消被跟踪的协程，但有一个现成的 `withTimeout` 函数可以完成此操作。 请看以下示例：


```kotlin
runBlocking {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}
```

    I'm sleeping 0 ...
    I'm sleeping 1 ...
    I'm sleeping 2 ...



    kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms

    	at _COROUTINE._BOUNDARY._(CoroutineDebugging.kt:42)

    	at Line_117_jupyter$1$1.invokeSuspend(Line_117.jupyter.kts:5) at Cell In[115], line 5

    	at Line_117_jupyter$1.invokeSuspend(Line_117.jupyter.kts:2) at Cell In[115], line 2

    Caused by: kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms

    	at kotlinx.coroutines.TimeoutKt.TimeoutCancellationException(Timeout.kt:189)

    	at kotlinx.coroutines.TimeoutCoroutine.run(Timeout.kt:157)

    	at kotlinx.coroutines.EventLoopImplBase$DelayedRunnableTask.run(EventLoop.common.kt:505)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.DefaultExecutor.run(DefaultExecutor.kt:105)

    	at java.base/java.lang.Thread.run(Unknown Source)

    

    kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms

    at Cell In[115], line 5


`withTimeout` 抛出的 `TimeoutCancellationException` 是 `CancellationException` 的子类。 我们之前没有在控制台上看到它的堆栈跟踪打印出来。那是因为 在已取消的协程内部，`CancellationException` 被认为是协程完成的正常原因。 但是，在此示例中，我们直接在 `main` 函数内部使用了 `withTimeout`。

由于取消只是一种异常，所有资源都以常规方式关闭。 如果你需要在任何类型的超时时执行一些额外的操作，可以将带有超时的代码包装在 `try {...} catch (e: TimeoutCancellationException) {...}` 代码块中，或者使用 `withTimeoutOrNull` 函数，它与 `withTimeout` 类似，但在超时时返回 `null` 而不是抛出异常：


```kotlin
runBlocking {
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // will get cancelled before it produces this result
    }
    println("Result is $result")
}
```

    I'm sleeping 0 ...
    I'm sleeping 1 ...
    I'm sleeping 2 ...
    Result is null


运行此代码时不再出现异常：

## 异步超时和资源

`withTimeout` 中的超时事件相对于其代码块中运行的代码是异步的，并且可能随时发生，甚至在超时代码块内部返回之前。如果你在代码块内部打开或获取了需要在代码块外部关闭或释放的资源，请记住这一点。

例如，这里我们用 `Resource` 类模拟一个可关闭资源，它通过递增 `acquired` 计数器来简单地跟踪其被创建的次数，并在其 `close` 函数中递减计数器。 现在让我们创建许多协程，每个协程在 `withTimeout` 块的末尾创建一个 `Resource`，并在块外部释放资源。我们添加一个小的延迟，以便超时更可能在 `withTimeout` 块已经完成时发生，这将导致资源泄漏。


```kotlin
var acquired = 0

class Resource {
    init {
        acquired++
    } // Acquire the resource

    fun close() {
        acquired--
    } // Release the resource
}

runBlocking {
    repeat(10_000) { // Launch 10K coroutines
        launch {
            val resource = withTimeout(60) { // Timeout of 60 ms
                delay(50) // Delay for 50 ms
                Resource() // Acquire a resource and return it from withTimeout block
            }
            resource.close() // Release the resource
        }
    }
}
// Outside of runBlocking all coroutines have completed
println(acquired) // Print the number of resources still acquired
```

    475


如果你运行上述代码，你会发现它不总是打印零，尽管这可能取决于你机器的时序。你可能需要调整此示例中的超时以实际看到非零值。

> 请注意，这里从 10K 个协程中递增和递减 `acquired` 计数器是完全线程安全的， 因为它总是发生在 `runBlocking` 使用的同一个线程中。 关于这方面的更多内容将在协程上下文一章中解释。

要解决此问题，你可以将对资源的引用存储在一个变量中，而不是从 `withTimeout` 代码块返回它。


```kotlin
var acquired = 0

class Resource {
    init {
        acquired++
    } // Acquire the resource

    fun close() {
        acquired--
    } // Release the resource
}

runBlocking {
    repeat(10_000) { // Launch 10K coroutines
        launch {
            var resource: Resource? = null // Not acquired yet
            try {
                withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    resource = Resource() // Store a resource to the variable if acquired
                }
                // We can do something else with the resource here
            } finally {
                resource?.close() // Release the resource if it was acquired
            }
        }
    }
}
// Outside of runBlocking all coroutines have completed
println(acquired) // Print the number of resources still acquired
```

    0


此示例始终打印零。资源不会泄漏。
