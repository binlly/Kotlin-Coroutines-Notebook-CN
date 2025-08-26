# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 协程异常处理

本节涵盖异常处理以及异常时的取消。 我们已经知道，已取消的协程会在挂起点抛出 [`CancellationException`]，并且会被协程机制忽略。这里我们看看如果在取消期间抛出异常，或者同一协程的多个子协程抛出异常时会发生什么。

异常传播
协程构建器有两种类型：自动传播异常 ([`launch`]) 或将它们暴露给用户 ([`async`] 和 [`produce`])。 当这些构建器用于创建不是其他协程的_子_协程的_根_协程时，前者会将异常视为未捕获异常，类似于 Java 的 `Thread.uncaughtExceptionHandler`，而后者则依赖用户消费最终的异常，例如通过 [`await`][Deferred.await] 或 [`receive`][ReceiveChannel.receive]（[`produce`] 和 [`receive`][ReceiveChannel.receive] 在 Channels 部分中介绍）。

这可以通过一个使用 [`GlobalScope`] 创建根协程的简单示例来演示：

> NOTE
> [`GlobalScope`] 是一个慎用的 API，可能会以复杂的方式产生意想不到的负面结果。为整个应用程序创建根协程是 `GlobalScope` 少数几个合理的使用场景之一，因此你必须通过 `@OptIn(DelicateCoroutinesApi::class)` 明确选择使用 `GlobalScope`。


```kotlin
@OptIn(DelicateCoroutinesApi::class)
runBlocking {
    val job = GlobalScope.launch { // root coroutine with launch
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Will be printed to the console by Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // root coroutine with async
        println("Throwing exception from async")
        throw ArithmeticException() // Nothing is printed, relying on user to call await
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

    Throwing exception from launch
    Joined failed job
    Throwing exception from async
    Caught ArithmeticException


## CoroutineExceptionHandler

可以自定义将未捕获异常打印到控制台的默认行为。_根_协程上的 [`CoroutineExceptionHandler`] 上下文元素可用作此根协程及其所有子协程的通用 `catch` 块，用于处理自定义异常。 它类似于 `Thread.uncaughtExceptionHandler`。 你无法在 `CoroutineExceptionHandler` 中从异常中恢复。当处理程序被调用时，协程已经因相应的异常而完成。通常，该处理程序用于记录异常、显示某种错误消息、终止和/或重启应用程序。

`CoroutineExceptionHandler` 仅对未捕获异常（即未以任何其他方式处理的异常）调用。 特别地，所有_子_协程（在另一个 [`Job`] 的上下文中创建的协程）都会将其异常处理委托给其父协程，后者又委托给其父协程，以此类推直到根协程，因此安装在它们上下文中的 `CoroutineExceptionHandler` 永远不会被使用。 此外，[`async`] 构建器总是捕获所有异常并在生成的 [`Deferred`] 对象中表示它们，因此其 `CoroutineExceptionHandler` 也无效。


```kotlin
@OptIn(DelicateCoroutinesApi::class)
runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) { // root coroutine, running in GlobalScope
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) { // also root, but async instead of launch
        throw ArithmeticException() // Nothing will be printed, relying on user to call deferred.await()
    }
    joinAll(job, deferred)
}
```

    CoroutineExceptionHandler got java.lang.AssertionError


## 取消与异常

取消与异常密切相关。协程内部使用 `CancellationException` 进行取消，这些异常被所有处理程序忽略，因此它们应该只用作额外调试信息的来源，可以通过 `catch` 块获取。 当协程使用 [`Job.cancel`] 取消时，它会终止，但不会取消其父级。


```kotlin
runBlocking {
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
}
```

    Cancelling child
    Child is cancelled
    Parent is not cancelled


如果协程遇到 `CancellationException` 以外的异常，它会用该异常取消其父级。此行为无法被覆盖，并用于为结构化并发提供稳定的协程层次结构。 [`CoroutineExceptionHandler`] 实现不用于子协程。

> NOTE
> 在这些示例中，[`CoroutineExceptionHandler`] 总是安装到在 [`GlobalScope`] 中创建的协程上。将异常处理程序安装到在主 [`runBlocking`] 范围内启动的协程是没有意义的，因为即使安装了处理程序，当其子协程因异常而完成时，主协程也总是会被取消。

只有当所有子协程终止时，父协程才会处理原始异常，这由以下示例演示。


```kotlin
@OptIn(DelicateCoroutinesApi::class)
runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        launch { // the first child
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // the second child
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
}
```

    Second child throws an exception
    Children are cancelled, but exception is not handled until all children terminate
    The first child finished its non cancellable block
    CoroutineExceptionHandler got java.lang.ArithmeticException


## 异常聚合

当一个协程的多个子协程因异常而失败时，一般规则是“第一个异常优先”，因此第一个异常得到处理。第一个异常之后发生的所有额外异常都作为被抑制的异常附加到第一个异常上。


```kotlin
import java.io.IOException

@OptIn(DelicateCoroutinesApi::class)
runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // it gets cancelled when another sibling fails with IOException
            } finally {
                throw ArithmeticException() // the second exception
            }
        }
        launch {
            delay(100)
            throw IOException() // the first exception
        }
        delay(Long.MAX_VALUE)
    }
    job.join()
}
```

    CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]


取消异常是透明的，默认会被解包：


```kotlin
@OptIn(DelicateCoroutinesApi::class)
runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        val innerJob = launch { // all this stack of coroutines will get cancelled
            launch {
                launch {
                    throw IOException() // the original exception
                }
                launch {
                    delay(100)
                    throw ArithmeticException()
                }
            }
        }
        try {
            innerJob.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e // cancellation exception is rethrown, yet the original IOException gets to the handler
        }
    }
    job.join()
}
```

    Rethrowing CancellationException with original cause
    CoroutineExceptionHandler got java.lang.ArithmeticException


## Supervisor

如前所述，取消是一种双向关系，在整个协程层次结构中传播。让我们来看看需要单向取消的情况。

这种要求的一个很好的例子是其作用域中定义了作业的 UI 组件。如果 UI 的任何子任务失败，并不总是需要取消（实际上是终止）整个 UI 组件，但如果 UI 组件被销毁（且其作业被取消），则需要取消所有子作业，因为它们的结果不再需要。

另一个例子是服务器进程，它产生多个子作业并需要_监管_它们的执行，跟踪它们的失败并只重启失败的作业。

## 监管作业

[`SupervisorJob`][SupervisorJob()] 可用于这些目的。它类似于普通的 [`Job`][Job()]，唯一的例外是取消仅向下传播。这可以通过以下示例轻松演示：


```kotlin
runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // launch the first child -- its exception is ignored for this example (don't do this in practice!)
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("The first child is failing")
            throw AssertionError("The first child is cancelled")
        }
        // launch the second child
        val secondChild = launch {
            firstChild.join()
            // Cancellation of the first child is not propagated to the second child
            println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // But cancellation of the supervisor is propagated
                println("The second child is cancelled because the supervisor was cancelled")
            }
        }
        // wait until the first child fails & completes
        firstChild.join()
        println("Cancelling the supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

    The first child is failing
    The first child is cancelled: true, but the second one is still active
    Cancelling the supervisor
    The second child is cancelled because the supervisor was cancelled


## Supervisor Scope

除了 [`coroutineScope`]] 之外，我们还可以使用 [`supervisorScope`] 实现_作用域_并发。它只在一个方向上传播取消，并且只有当自身失败时才取消所有子协程。它也像 [`coroutineScope`] 一样，在完成之前等待所有子协程。


```kotlin
runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("The child is cancelled")
                }
            }
            // Give our child a chance to execute and print using yield
            yield()
            println("Throwing an exception from the scope")
            throw AssertionError()
        }
    } catch (e: AssertionError) {
        println("Caught an assertion error")
    }
}
```

    The child is sleeping
    Throwing an exception from the scope
    The child is cancelled
    Caught an assertion error


## 监管协程中的异常

常规作业和监管作业之间的另一个关键区别是异常处理。 每个子协程都应该通过异常处理机制自己处理其异常。 这种差异源于子协程的失败不会传播给父协程。 这意味着直接在 [`supervisorScope`] 内启动的协程_确实_使用安装在其作用域中的 [`CoroutineExceptionHandler`]，与根协程的方式相同（详见 `CoroutineExceptionHandler` 部分）。


```kotlin
runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    supervisorScope {
        val child = launch(handler) {
            println("The child throws an exception")
            throw AssertionError()
        }
        println("The scope is completing")
    }
    println("The scope is completed")
}
```

    The scope is completing
    The child throws an exception
    CoroutineExceptionHandler got java.lang.AssertionError
    The scope is completed

