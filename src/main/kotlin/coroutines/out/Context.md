# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 协程上下文与调度器

协程总是在某个上下文中执行，该上下文由 `Kotlin` 标准库中定义的 `CoroutineContext` 类型的值表示。

协程上下文是各种元素的集合。主要元素是协程的 `Job`（我们之前已经见过）及其调度器，本节将介绍它。

## 调度器与线程

协程上下文包含一个 协程调度器（参见 `CoroutineDispatcher`），它决定了相应协程用于执行的线程或线程集。协程调度器可以将协程执行限制在特定线程、将其分派到线程池，或者让其以非受限方式运行。

所有协程构建器（如 `launch` 和 `async`）都接受一个可选的 `CoroutineContext` 参数，该参数可用于显式指定新协程的调度器及其他上下文元素。

尝试以下示例：


```kotlin
runBlocking<Unit> {
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}
```

    Unconfined            : I'm working in thread Execution of code 'runBlocking<Unit> {...' @coroutine#3
    Default               : I'm working in thread DefaultDispatcher-worker-1 @coroutine#4
    newSingleThreadContext: I'm working in thread MyOwnThread @coroutine#5
    main runBlocking      : I'm working in thread Execution of code 'runBlocking<Unit> {...' @coroutine#2


当 `launch { ... }` 不带参数使用时，它会从其启动的 `CoroutineScope` 继承上下文（以及调度器）。在这种情况下，它继承了运行在 `main` 线程中的主 `runBlocking` 协程的上下文。

`Dispatchers.Unconfined` 是一种特殊的调度器，它似乎也在 `main` 线程中运行，但实际上，它是一种不同的机制，稍后会进行解释。

当作用域中未显式指定其他调度器时，会使用默认调度器。它由 `Dispatchers.Default` 表示，并使用一个共享的后台线程池。

`newSingleThreadContext` 会为协程创建一个线程来运行。一个专用线程是非常昂贵的资源。在实际应用程序中，当不再需要时，必须使用 `close` 函数释放它，或者将其存储在顶层变量中并在整个应用程序中重复使用。



## 非受限调度器与受限调度器

`Dispatchers.Unconfined` 协程调度器会在调用者线程中启动协程，但仅限于第一个挂起点之前。挂起之后，它会在完全由所调用的挂起函数确定的线程中恢复协程。非受限调度器适用于那些既不消耗 `CPU` 时间，也不更新任何限制在特定线程上的共享数据（如 UI）的协程。

另一方面，调度器默认从外部 `CoroutineScope` 继承。特别是，`runBlocking` 协程的默认调度器受限于调用者线程，因此继承它会使执行受限于该线程，并具有可预测的 `FIFO` 调度。


```kotlin
runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
}
```

    Unconfined      : I'm working in thread Execution of code 'runBlocking<Unit> {...' @coroutine#7
    main runBlocking: I'm working in thread Execution of code 'runBlocking<Unit> {...' @coroutine#8
    Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor @coroutine#7
    main runBlocking: After delay in thread Execution of code 'runBlocking<Unit> {...' @coroutine#8


## 线程间跳转
使用 `-Dkotlinx.coroutines.debug JVM` 选项运行以下代码（参见调试）：


```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

newSingleThreadContext("Ctx1").use { ctx1 ->
    newSingleThreadContext("Ctx2").use { ctx2 ->
        runBlocking(ctx1) {
            log("Started in ctx1")
            withContext(ctx2) {
                log("Working in ctx2")
            }
            log("Back to ctx1")
        }
    }
}
```

    [Ctx1 @coroutine#9] Started in ctx1
    [Ctx2 @coroutine#9] Working in ctx2
    [Ctx1 @coroutine#9] Back to ctx1


上面的示例展示了协程使用中的新技巧。

第一种技巧展示了如何使用带指定上下文的 `runBlocking`。第二种技巧涉及调用 `withContext`，它可能会挂起当前协程并切换到新上下文——前提是新上下文与现有上下文不同。具体来说，如果你指定了不同的 `CoroutineDispatcher`，则需要额外的分派：该代码块会在新调度器上调度，一旦完成，执行将返回到原始调度器。

> 上面的示例使用了 `Kotlin` 标准库中的 `use` 函数，以便在不再需要时正确释放由 `newSingleThreadContext` 创建的线程资源。

## 上下文中的 Job

协程的 `Job` 是其上下文的一部分，可以使用 `coroutineContext[Job]` 表达式从中检索：


```kotlin
runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")
}
```

    My job is "coroutine#10":BlockingCoroutine{Active}@6bb7d2cc


> 请注意，`CoroutineScope` 中的 `isActive` 只是 `coroutineContext[Job]?.isActive == true` 的一个便捷快捷方式。

## 协程的子协程

当一个协程在另一个协程的 `CoroutineScope` 中启动时，它会通过 `CoroutineScope.coroutineContext` 继承其上下文，并且新协程的 `Job` 会成为父协程 `Job` 的_子级_。当父协程被取消时，它的所有子协程也会被递归取消。

然而，这种父子关系可以通过以下两种方式之一显式覆盖：

当启动协程时显式指定不同的作用域（例如 `GlobalScope.launch`）时，它不会从父作用域继承 `Job`。
当一个不同的 `Job` 对象作为新协程的上下文传递时（如下例所示），它会覆盖父作用域的 `Job`。
在这两种情况下，启动的协程都不会绑定到其启动的作用域，并独立运行。


```kotlin
runBlocking<Unit> {
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        // it spawns two other jobs
        launch(Job()) {
            println("job1: I run in my own Job and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    delay(500)
    request.cancel() // cancel processing of the request
    println("main: Who has survived request cancellation?")
    delay(1000) // delay the main thread for a second to see what happens
}
```

    job1: I run in my own Job and execute independently!
    job2: I am a child of the request coroutine
    main: Who has survived request cancellation?
    job1: I am not affected by cancellation of the request


## 父级职责

父协程总是等待其所有子协程的完成。父级不必显式跟踪它启动的所有子协程，也无需在最后使用 `Job.join` 等待它们：


```kotlin
runBlocking<Unit> {
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        repeat(3) { i -> // launch a few children jobs
            launch  {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() // wait for completion of the request, including all its children
    println("Now processing of the request is complete")
}
```

    request: I'm done and I don't explicitly join my children that are still active
    Coroutine 0 is done
    Coroutine 1 is done
    Coroutine 2 is done
    Now processing of the request is complete


## 组合上下文元素

有时我们需要为一个协程上下文定义多个元素。我们可以使用 `+` 运算符来实现。例如，我们可以同时启动一个协程，并显式指定调度器和名称：


```kotlin
runBlocking<Unit> {
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
}
```

    I'm working in thread DefaultDispatcher-worker-1 @test#21


## 线程局部数据


```kotlin
val threadLocal = ThreadLocal<String?>() // declare thread-local variable

runBlocking<Unit> {
    threadLocal.set("父协程")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "子协程")) {
        println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
}
```

    Pre-main, current thread: Thread[#30,Execution of code 'val threadLocal = Th...' @coroutine#22,5,main], thread local value: '父协程'
    Launch start, current thread: Thread[#92,DefaultDispatcher-worker-1 @coroutine#23,5,main], thread local value: '子协程'
    After yield, current thread: Thread[#92,DefaultDispatcher-worker-1 @coroutine#23,5,main], thread local value: '子协程'
    Post-main, current thread: Thread[#30,Execution of code 'val threadLocal = Th...' @coroutine#22,5,main], thread local value: '父协程'

