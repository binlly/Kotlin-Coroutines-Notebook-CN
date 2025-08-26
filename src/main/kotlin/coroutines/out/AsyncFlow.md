# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 流

## 流构建器

各种集合和序列都可以使用 `.asFlow()` 扩展函数转换为流：


```kotlin
runBlocking<Unit> {
    // Convert an integer range to a flow
    (1..3).asFlow().collect { value -> println(value) }
}
```

    1
    2
    3


## 中间流操作符

流可以使用操作符进行转换，就像你转换集合和序列一样。中间操作符应用于上游流并返回下游流。这些操作符是冷的，就像流一样。调用这样的操作符本身不是一个挂起函数。它工作快速，返回一个新的转换流的定义。

基本操作符有像 `map` 和 `filter` 这样熟悉的名字。这些操作符与序列的一个重要区别是，这些操作符内部的代码块可以调用挂起函数。

例如，传入请求的流可以使用 `map` 操作符映射到其结果，即使执行请求是一个由挂起函数实现的长期运行操作：


```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
```

    response 1
    response 2
    response 3


## Transform 操作符

在流转换操作符中，最通用的一种是 `transform`。它既可以用于模拟 `map` 和 `filter` 等简单转换，也可以实现更复杂的转换。使用 `transform` 操作符，我们可以任意次数地 发出 任意值。

例如，使用 `transform`，我们可以在执行长时间运行的异步请求之前发出一个字符串，然后接着发出一个响应：


```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .transform { request ->
            emit("Making request $request")
            emit(performRequest(request))
        }
        .collect { response -> println(response) }
}
```

    Making request 1
    response 1
    Making request 2
    response 2
    Making request 3
    response 3


## 尺寸限制操作符

take 等尺寸限制的中间操作符在达到相应的限制时会取消流的执行。协程中的取消总是通过抛出异常来执行，这样所有资源管理函数（例如 `try { ... } finally { ... }` 块）在取消时都能正常运行：


```kotlin
fun numbers(): Flow<Int> = flow {
    try {
        emit(1)
        emit(2)
        println("This line will not execute")
        emit(3)
    } finally {
        println("Finally in numbers")
    }
}

runBlocking<Unit> {
    numbers()
        .take(2) // take only the first two
        .collect { value -> println(value) }
}
```

    1
    2
    Finally in numbers


## 终止流操作符

流上的终止操作符是 挂起函数，它们启动流的收集。 `collect` 操作符是最基本的一个，但还有其他终止操作符，可以使其更容易：

转换为各种集合，如 `toList` 和 `toSet`。
获取第一个值 (`first`) 并确保流发出单个值 (`single`) 的操作符。
使用 `reduce` 和 `fold` 将流归约为一个值。
例如：


```kotlin
runBlocking<Unit> {
    val sum = (1..5).asFlow()
        .map { it * it } // squares of numbers from 1 to 5
        .reduce { a, b -> a + b } // sum them (terminal operator)
    println(sum)
}
```

    55


## 流是顺序的 ([Flows are sequential])

流的每次独立收集都是顺序执行的，除非使用作用于多个流的特殊操作符。收集直接在调用终止操作符的协程中工作。默认情况下不会启动新的协程。每个发出的值都会被所有中间操作符从上游到下游处理，然后传递给终止操作符。

请看以下示例，它过滤偶数整数并将它们映射为字符串：


```kotlin
runBlocking<Unit> {
    (1..5).asFlow()
        .filter {
            println("Filter $it")
            it % 2 == 0
        }
        .map {
            println("Map $it")
            "string $it"
        }.collect {
            println("Collect $it")
        }
}
```

    Filter 1
    Filter 2
    Map 2
    Collect string 2
    Filter 3
    Filter 4
    Map 4
    Collect string 4
    Filter 5


## flowOn 操作符

异常指的是 `flowOn` 函数，该函数应用于改变流发出的上下文。改变流上下文的正确方法如下面示例所示，它还打印相应线程的名称以展示其工作原理：


```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it in CPU-consuming way
        log("Emitting $i")
        emit(i) // emit next value
    }
}.flowOn(Dispatchers.Default) // RIGHT way to change context for CPU-consuming code in flow builder

runBlocking<Unit> {
    simple().collect { value ->
        log("Collected $value")
    }
}
```

    [DefaultDispatcher-worker-1 @coroutine#136] Emitting 1
    [Execution of code 'fun log(msg: String)...' @coroutine#135] Collected 1
    [DefaultDispatcher-worker-1 @coroutine#136] Emitting 2
    [Execution of code 'fun log(msg: String)...' @coroutine#135] Collected 2
    [DefaultDispatcher-worker-1 @coroutine#136] Emitting 3
    [Execution of code 'fun log(msg: String)...' @coroutine#135] Collected 3


## 缓冲

在不同的协程中运行流的不同部分有助于缩短收集流的总时间，特别是在涉及长期运行的异步操作时。例如，考虑一种情况，`simple` 流的发出很慢，产生一个元素需要 100 毫秒；并且收集器也很慢，处理一个元素需要 300 毫秒。让我们看看收集这样一个包含三个数字的流需要多长时间：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

runBlocking<Unit> {
    val time = measureTimeMillis {
        simple().collect { value ->
            delay(300) // pretend we are processing it for 300 ms
            println(value)
        }
    }
    println("Collected in $time ms")
}
```

    1
    2
    3
    Collected in 1225 ms


我们可以在流上使用 `buffer` 操作符，让 `simple` 流的发出代码与收集代码并发运行，而不是顺序运行它们：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .buffer() // buffer emissions, don't wait
            .collect { value ->
                delay(300) // pretend we are processing it for 300 ms
                println(value)
            }
    }
    println("Collected in $time ms")
}
```

    1
    2
    3
    Collected in 1022 ms


## 合并 ([Conflation])

当流表示操作的部分结果或操作状态更新时，可能不需要处理每个值，而只需要处理最新值。在这种情况下，可以使用 `conflate` 操作符来跳过中间值，当收集器处理它们的速度太慢时。基于之前的示例：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .conflate() // conflate emissions, don't process each one
            .collect { value ->
                delay(300) // pretend we are processing it for 300 ms
                println(value)
            }
    }
    println("Collected in $time ms")
}
```

    1
    3
    Collected in 714 ms


我们看到，当第一个数字仍在处理时，第二个和第三个数字已经生成，因此第二个数字被 合并 了，只有最新（第三个）的数字被传递给收集器：

## 处理最新值

合并是加速处理的一种方式，当发出者和收集器都很慢时。它通过丢弃发出的值来实现。另一种方法是取消一个慢速收集器，并在每次发出新值时重新启动它。有一系列 `xxxLatest` 操作符，它们执行 `xxx` 操作符相同的基本逻辑，但**_在新值出现时取消其块中的代码_**。让我们尝试将 `conflate` 更改为 `collectLatest` 在之前的示例中：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // cancel & restart on the latest value
                println("Collecting $value")
                delay(300) // pretend we are processing it for 300 ms
                println("Done $value")
            }
    }
    println("Collected in $time ms")
}
```

    Collecting 1
    Collecting 2
    Collecting 3
    Done 3
    Collected in 614 ms


## 组合多个流

## Zip


```kotlin
runBlocking<Unit> {
    val nums = (1..3).asFlow() // numbers 1..3
    val strs = flowOf("one", "two", "three") // strings
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string
        .collect { println(it) } // collect and print
}
```

    1 -> one
    2 -> two
    3 -> three


## Combine

当流表示变量或操作的最新值时（另请参阅有关 合并 的相关部分），可能需要执行一个依赖于相应流最新值的计算，并且每当任何上游流发出值时重新计算它。相应的操作符系列称为 `combine`。

例如，如果前面示例中的数字每 `300` 毫秒更新一次，但字符串每 `400` 毫秒更新一次，那么使用 `zip` 操作符将它们打包仍将产生相同的结果，尽管结果每 `400` 毫秒打印一次：


```kotlin
runBlocking<Unit> {
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
    val startTime = System.currentTimeMillis() // remember the start time
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string with "zip"
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

    1 -> one at 406 ms from start
    2 -> two at 813 ms from start
    3 -> three at 1218 ms from start


然而，当这里使用 `combine` 操作符而不是 `zip` 时：


```kotlin
runBlocking<Unit> {
    val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
    val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
    val startTime = System.currentTimeMillis() // remember the start time
    nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

    1 -> one at 406 ms from start
    2 -> one at 609 ms from start
    2 -> two at 811 ms from start
    3 -> two at 914 ms from start
    3 -> three at 1212 ms from start


## 展平流

流表示异步接收到的值序列，因此很容易出现每个值都触发对另一个值序列请求的情况。例如，我们可以有以下函数，它返回一个包含两个字符串的流，间隔 500 毫秒：


```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}
```

现在，如果我们有一个包含三个整数的流，并对它们中的每一个调用 `requestFlow`，像这样：


```kotlin
(1..3).asFlow().map { requestFlow(it) }
```




    Line_84_jupyter$special$$inlined$map$1@4457a1f5



那么我们最终会得到一个流的流（`Flow<Flow<String>>`），需要将其 展平 成一个单独的流以供进一步处理。集合和序列有 `flatten` 和 `flatMap` 操作符来完成此操作。然而，由于流的异步特性，它们需要不同的展平 模式，因此存在一系列流的展平操作符。

## flatMapConcat

流的流的连接由 `flatMapConcat` 和 `flattenConcat` 操作符提供。它们是相应序列操作符最直接的类比。它们等待内部流完成后才开始收集下一个流，如下面示例所示：


```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

runBlocking<Unit> {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // emit a number every 100 ms
        .flatMapConcat { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

    1: First at 107 ms from start
    1: Second at 607 ms from start
    2: First at 712 ms from start
    2: Second at 1217 ms from start
    3: First at 1323 ms from start
    3: Second at 1828 ms from start


## flatMapMerge

另一种展平操作是并发收集所有传入的流，并将它们的值合并到一个流中，以便值尽快发出。它由 `flatMapMerge` 和 `flattenMerge` 操作符实现。它们都接受一个可选的 `concurrency` 参数，该参数限制了同时收集的并发流的数量（默认情况下它等于 `DEFAULT_CONCURRENCY`）。


```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

runBlocking<Unit> {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapMerge { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

    1: First at 106 ms from start
    2: First at 206 ms from start
    3: First at 309 ms from start
    1: Second at 607 ms from start
    2: Second at 711 ms from start
    3: Second at 814 ms from start


> NOTE

> 请注意，flatMapMerge 按顺序调用其代码块（在此示例中为 { `requestFlow(it)` }），但并发收集结果流，这相当于首先执行顺序的 `map { requestFlow(it) }`，然后对结果调用 `flattenMerge`。

## flatMapLatest

与 `collectLatest` 操作符类似，`collectLatest` 操作符在“处理最新值”部分中有所描述，存在相应的“最新”展平模式，其中一旦发出新流，前一个流的收集就会被取消。它由 `flatMapLatest` 操作符实现。


```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

runBlocking<Unit> {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapLatest { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

    1: First at 106 ms from start
    2: First at 208 ms from start
    3: First at 313 ms from start
    3: Second at 818 ms from start


## 流异常

## 收集器的 try 和 catch


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

runBlocking<Unit> {
    try {
        simple().collect { value ->
            println(value)
            check(value <= 1) { "异常值： $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}
```

    Emitting 1
    1
    Emitting 2
    2
    Caught java.lang.IllegalStateException: 异常值： 2


## 一切都被捕获

前面的示例实际上捕获了发出者或任何中间或终止操作符中发生的任何异常。例如，让我们更改代码，以便发出的值被映射为字符串，但相应的代码产生异常：（此异常仍然被捕获，收集被停止）


```kotlin

```


```kotlin
fun simple(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }.map { value ->
        check(value <= 1) { "Crashed on $value" }
        "string $value"
    }

runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}
```

    Emitting 1
    string 1
    Emitting 2
    Caught java.lang.IllegalStateException: Crashed on 2


## 异常透明性

但是发出者的代码如何封装其异常处理行为呢？

流必须对异常是 透明的，在 `flow { ... }` 构建器内部的 `try/catch` 块中 发出 值是违反异常透明性的。这保证了抛出异常的收集器总能像前面的示例一样使用 `try/catch` 捕获它。

发出者可以使用 `catch` 操作符，该操作符保留了这种异常透明性并允许封装其异常处理。`catch` 操作符的主体可以分析异常并根据捕获到的异常以不同的方式对其作出反应：

异常可以使用 `throw` 重新抛出。
异常可以通过 `catch` 主体中的 `emit` 转换为值的发出。
异常可以被忽略、记录或由其他代码处理。
例如，让我们在捕获到异常时发出文本：


```kotlin
fun simple(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
        .map { value ->
            check(value <= 1) { "Crashed on $value" }
            "string $value"
        }

runBlocking<Unit> {
    simple()
        .catch { e -> emit("Caught $e") } // emit on exception
        .collect { value -> println(value) }
}
```

    Emitting 1
    string 1
    Emitting 2
    Caught java.lang.IllegalStateException: Crashed on 2


## 透明捕获

`catch` 中间操作符，遵循异常透明性，只捕获上游异常（即来自 `catch` 上方所有操作符的异常，而不是其下方）。如果 `collect { ... }` 中的块（位于 `catch` 下方）抛出异常，那么它会逃逸：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

runBlocking<Unit> {
    simple()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
}
```

    Emitting 1
    1
    Emitting 2



    java.lang.IllegalStateException: Collected 2

    	at Line_91_jupyter$1$2.emit(Line_91.jupyter.kts:12) at Cell In[90], line 12

    	at Line_91_jupyter$1$2.emit(Line_91.jupyter.kts:11) at Cell In[90], line 11

    	at kotlinx.coroutines.flow.FlowKt__ErrorsKt$catchImpl$2.emit(Errors.kt:154)

    	at kotlinx.coroutines.flow.internal.SafeCollectorKt$emitFun$1.invoke(SafeCollector.kt:11)

    	at kotlinx.coroutines.flow.internal.SafeCollectorKt$emitFun$1.invoke(SafeCollector.kt:11)

    	at kotlinx.coroutines.flow.internal.SafeCollector.emit(SafeCollector.kt:113)

    	at kotlinx.coroutines.flow.internal.SafeCollector.emit(SafeCollector.kt:82)

    	at Line_91_jupyter$simple$1.invokeSuspend(Line_91.jupyter.kts:4) at Cell In[90], line 4

    	at Line_91_jupyter$simple$1.invoke(Line_91.jupyter.kts)

    	at Line_91_jupyter$simple$1.invoke(Line_91.jupyter.kts)

    	at kotlinx.coroutines.flow.SafeFlow.collectSafely(Builders.kt:57)

    	at kotlinx.coroutines.flow.AbstractFlow.collect(Flow.kt:226)

    	at kotlinx.coroutines.flow.FlowKt__ErrorsKt.catchImpl(Errors.kt:152)

    	at kotlinx.coroutines.flow.FlowKt.catchImpl(Unknown Source)

    	at kotlinx.coroutines.flow.FlowKt__ErrorsKt$catch$$inlined$unsafeFlow$1.collect(SafeCollector.common.kt:109)

    	at Line_91_jupyter$1.invokeSuspend(Line_91.jupyter.kts:11) at Cell In[90], line 11

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:94)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:70)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:48)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at Line_91_jupyter.<init>(Line_91.jupyter.kts:8) at Cell In[90], line 8

    	at java.base/jdk.internal.reflect.DirectConstructorHandleAccessor.newInstance(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstance(Unknown Source)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.evalWithConfigAndOtherScriptsResults(BasicJvmScriptEvaluator.kt:122)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke$suspendImpl(BasicJvmScriptEvaluator.kt:48)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke(BasicJvmScriptEvaluator.kt)

    	at kotlin.script.experimental.jvm.BasicJvmReplEvaluator.eval(BasicJvmReplEvaluator.kt:49)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl$eval$resultWithDiagnostics$1.invokeSuspend(InternalEvaluatorImpl.kt:138)

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl.eval(InternalEvaluatorImpl.kt:138)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute_L4Nmkdk$lambda$9$lambda$1(CellExecutorImpl.kt:80)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withHost(ReplForJupyterImpl.kt:794)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute-L4Nmkdk(CellExecutorImpl.kt:78)

    	at org.jetbrains.kotlinx.jupyter.repl.execution.CellExecutor.execute-L4Nmkdk$default(CellExecutor.kt:14)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evaluateUserCode-wNURfNM(ReplForJupyterImpl.kt:616)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalExImpl(ReplForJupyterImpl.kt:474)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx$lambda$20(ReplForJupyterImpl.kt:467)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withEvalContext(ReplForJupyterImpl.kt:447)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx(ReplForJupyterImpl.kt:466)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6$lambda$5(IdeCompatibleMessageRequestProcessor.kt:160)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedIn(IdeCompatibleMessageRequestProcessor.kt:354)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16$lambda$15(IdeCompatibleMessageRequestProcessor.kt:368)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedErr(IdeCompatibleMessageRequestProcessor.kt:343)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16(IdeCompatibleMessageRequestProcessor.kt:367)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedOut(IdeCompatibleMessageRequestProcessor.kt:335)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO(IdeCompatibleMessageRequestProcessor.kt:366)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6(IdeCompatibleMessageRequestProcessor.kt:159)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl$Task.execute(JupyterExecutorImpl.kt:41)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl.executorThread$lambda$0(JupyterExecutorImpl.kt:83)

    	at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

    

    java.lang.IllegalStateException: Collected 2

    at Cell In[90], line 12


## 声明式捕获

我们可以结合 `catch` 操作符的声明性与处理所有异常的愿望，方法是将 `collect` 操作符的主体移到 onEach 中，并将其放在 `catch` 操作符之前。此流的收集必须通过调用不带参数的 `collect()` 来触发：


```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

runBlocking<Unit> {
    simple()
        .onEach { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
        .catch { e -> println("Caught $e") }
        .collect()
}
```

    Emitting 1
    1
    Emitting 2
    Caught java.lang.IllegalStateException: Collected 2


## 流完成

当流收集完成（正常或异常）时，可能需要执行一个操作。正如您可能已经注意到的，这可以通过两种方式完成：命令式或声明式。

## 命令式 finally 块

除了 `try/catch`，收集器还可以使用 `finally` 块在 `collect` 完成时执行一个操作。


```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}
```

    1
    2
    3
    Done


## 声明式处理

对于声明式方法，流有一个 `onCompletion` 中间操作符，当流完全收集完毕时被调用。

前面的示例可以使用 `onCompletion` 操作符重写，并产生相同的输出：


```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

runBlocking<Unit> {
    simple()
        .onCompletion { println("Done") }
        .collect { value -> println(value) }
}
```

    1
    2
    3
    Done


`onCompletion` 的关键优势是 `lambda` 的可空 `Throwable` 参数，它可以用于确定流收集是正常完成还是异常完成。在以下示例中，`simple` 流在发出数字 `1` 后抛出异常：


```kotlin
fun simple(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

runBlocking<Unit> {
    simple()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}
```

    1
    Flow completed exceptionally
    Caught exception


`onCompletion` 操作符与 `catch` 不同，它不处理异常。正如我们从上面的示例代码中看到的，异常仍然流向下游。它将被传递给进一步的 `onCompletion` 操作符，并且可以用 `catch` 操作符处理。

## 成功完成

与 `catch` 操作符的另一个区别是，`onCompletion` 看到所有异常，并且仅在上游流成功完成时（没有取消或失败）接收 `null` 异常。


```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

runBlocking<Unit> {
    simple()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
}
```

    1
    Flow completed with java.lang.IllegalStateException: Collected 2



    java.lang.IllegalStateException: Collected 2

    	at Line_51_jupyter$1$2.emit(Line_51.jupyter.kts:7) at Cell In[50], line 7

    	at Line_51_jupyter$1$2.emit(Line_51.jupyter.kts:6) at Cell In[50], line 6

    	at kotlinx.coroutines.flow.FlowKt__BuildersKt$asFlow$$inlined$unsafeFlow$9.collect(SafeCollector.common.kt:111)

    	at kotlinx.coroutines.flow.FlowKt__EmittersKt$onCompletion$$inlined$unsafeFlow$1.collect(SafeCollector.common.kt:110)

    	at Line_51_jupyter$1.invokeSuspend(Line_51.jupyter.kts:6) at Cell In[50], line 6

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:94)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:70)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:48)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at Line_51_jupyter.<init>(Line_51.jupyter.kts:3) at Cell In[50], line 3

    	at java.base/jdk.internal.reflect.DirectConstructorHandleAccessor.newInstance(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstance(Unknown Source)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.evalWithConfigAndOtherScriptsResults(BasicJvmScriptEvaluator.kt:122)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke$suspendImpl(BasicJvmScriptEvaluator.kt:48)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke(BasicJvmScriptEvaluator.kt)

    	at kotlin.script.experimental.jvm.BasicJvmReplEvaluator.eval(BasicJvmReplEvaluator.kt:49)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl$eval$resultWithDiagnostics$1.invokeSuspend(InternalEvaluatorImpl.kt:138)

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl.eval(InternalEvaluatorImpl.kt:138)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute_L4Nmkdk$lambda$9$lambda$1(CellExecutorImpl.kt:80)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withHost(ReplForJupyterImpl.kt:794)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute-L4Nmkdk(CellExecutorImpl.kt:78)

    	at org.jetbrains.kotlinx.jupyter.repl.execution.CellExecutor.execute-L4Nmkdk$default(CellExecutor.kt:14)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evaluateUserCode-wNURfNM(ReplForJupyterImpl.kt:616)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalExImpl(ReplForJupyterImpl.kt:474)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx$lambda$20(ReplForJupyterImpl.kt:467)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withEvalContext(ReplForJupyterImpl.kt:447)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx(ReplForJupyterImpl.kt:466)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6$lambda$5(IdeCompatibleMessageRequestProcessor.kt:160)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedIn(IdeCompatibleMessageRequestProcessor.kt:354)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16$lambda$15(IdeCompatibleMessageRequestProcessor.kt:368)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedErr(IdeCompatibleMessageRequestProcessor.kt:343)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16(IdeCompatibleMessageRequestProcessor.kt:367)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedOut(IdeCompatibleMessageRequestProcessor.kt:335)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO(IdeCompatibleMessageRequestProcessor.kt:366)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6(IdeCompatibleMessageRequestProcessor.kt:159)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl$Task.execute(JupyterExecutorImpl.kt:41)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl.executorThread$lambda$0(JupyterExecutorImpl.kt:83)

    	at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

    

    java.lang.IllegalStateException: Collected 2

    at Cell In[50], line 7


## 启动流

很容易使用流来表示来自某个源的异步事件。在这种情况下，我们需要一个类似于 `addEventListener` 函数的类比，它注册一段代码以响应传入事件并继续后续工作。`onEach` 操作符可以扮演这个角色。然而，`onEach` 是一个中间操作符。我们还需要一个终止操作符来收集流。否则，仅仅调用 `onEach` 没有效果。

如果我们在 `onEach` 之后使用 `collect` 终止操作符，那么它后面的代码将等到流被收集为止：


```kotlin
// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .collect() // <--- Collecting the flow waits
    println("Done")
}
```

    Event: 1
    Event: 2
    Event: 3
    Done


`launchIn` 终止操作符在这里派上用场。通过将 `collect` 替换为 `launchIn`，我们可以在单独的协程中启动流的收集，这样后续代码的执行立即继续：


```kotlin
// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- Launching the flow in a separate coroutine
    println("Done")
}
```

    Done
    Event: 1
    Event: 2
    Event: 3


`launchIn` 的必需参数必须指定一个 `CoroutineScope`，用于启动收集流的协程。在上面的示例中，这个作用域来自 `runBlocking` 协程构建器，因此当流运行时，这个 `runBlocking` 作用域会等待其子协程完成，并阻止 `main` 函数返回并终止此示例。

在实际应用中，作用域将来自具有有限生命周期的实体。一旦此实体的生命周期终止，相应的作用域就会被取消，从而取消相应流的收集。这样，`onEach { ... }.launchIn(scope)` 组合就像 `addEventListener` 一样工作。然而，不需要相应的 `removeEventListener` 函数，因为取消和结构化并发可以达到这个目的。

请注意，`launchIn` 也返回一个 `Job`，它可以用于仅 取消 相应的流收集协程，而不取消整个作用域，或者 `join` 它。

## 流取消检查

为了方便，`flow` 构建器对每个发出的值执行额外的 `ensureActive` 取消检查。这意味着从 `flow { ... }` 发出值的繁忙循环是可取消的：


```kotlin
fun foo(): Flow<Int> = flow {
    for (i in 1..5) {
        println("Emitting $i")
        emit(i)
    }
}

runBlocking<Unit> {
    foo().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
```

    Emitting 1
    1
    Emitting 2
    2
    Emitting 3
    3
    Emitting 4



    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#89":BlockingCoroutine{Cancelled}@72367681

    	at kotlinx.coroutines.JobSupport.cancel(JobSupport.kt:1685)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel(CoroutineScope.kt:309)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel$default(CoroutineScope.kt:307)

    	at Line_54_jupyter$1$1.emit(Line_54.jupyter.kts:10) at Cell In[53], line 10

    	at Line_54_jupyter$1$1.emit(Line_54.jupyter.kts:9) at Cell In[53], line 9

    	at kotlinx.coroutines.flow.internal.SafeCollectorKt$emitFun$1.invoke(SafeCollector.kt:11)

    	at kotlinx.coroutines.flow.internal.SafeCollectorKt$emitFun$1.invoke(SafeCollector.kt:11)

    	at kotlinx.coroutines.flow.internal.SafeCollector.emit(SafeCollector.kt:113)

    	at kotlinx.coroutines.flow.internal.SafeCollector.emit(SafeCollector.kt:82)

    	at Line_54_jupyter$foo$1.invokeSuspend(Line_54.jupyter.kts:4) at Cell In[53], line 4

    	at Line_54_jupyter$foo$1.invoke(Line_54.jupyter.kts)

    	at Line_54_jupyter$foo$1.invoke(Line_54.jupyter.kts)

    	at kotlinx.coroutines.flow.SafeFlow.collectSafely(Builders.kt:57)

    	at kotlinx.coroutines.flow.AbstractFlow.collect(Flow.kt:226)

    	at Line_54_jupyter$1.invokeSuspend(Line_54.jupyter.kts:9) at Cell In[53], line 9

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:94)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:70)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:48)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at Line_54_jupyter.<init>(Line_54.jupyter.kts:8) at Cell In[53], line 8

    	at java.base/jdk.internal.reflect.DirectConstructorHandleAccessor.newInstance(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstance(Unknown Source)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.evalWithConfigAndOtherScriptsResults(BasicJvmScriptEvaluator.kt:122)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke$suspendImpl(BasicJvmScriptEvaluator.kt:48)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke(BasicJvmScriptEvaluator.kt)

    	at kotlin.script.experimental.jvm.BasicJvmReplEvaluator.eval(BasicJvmReplEvaluator.kt:49)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl$eval$resultWithDiagnostics$1.invokeSuspend(InternalEvaluatorImpl.kt:138)

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl.eval(InternalEvaluatorImpl.kt:138)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute_L4Nmkdk$lambda$9$lambda$1(CellExecutorImpl.kt:80)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withHost(ReplForJupyterImpl.kt:794)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute-L4Nmkdk(CellExecutorImpl.kt:78)

    	at org.jetbrains.kotlinx.jupyter.repl.execution.CellExecutor.execute-L4Nmkdk$default(CellExecutor.kt:14)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evaluateUserCode-wNURfNM(ReplForJupyterImpl.kt:616)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalExImpl(ReplForJupyterImpl.kt:474)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx$lambda$20(ReplForJupyterImpl.kt:467)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withEvalContext(ReplForJupyterImpl.kt:447)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx(ReplForJupyterImpl.kt:466)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6$lambda$5(IdeCompatibleMessageRequestProcessor.kt:160)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedIn(IdeCompatibleMessageRequestProcessor.kt:354)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16$lambda$15(IdeCompatibleMessageRequestProcessor.kt:368)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedErr(IdeCompatibleMessageRequestProcessor.kt:343)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16(IdeCompatibleMessageRequestProcessor.kt:367)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedOut(IdeCompatibleMessageRequestProcessor.kt:335)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO(IdeCompatibleMessageRequestProcessor.kt:366)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6(IdeCompatibleMessageRequestProcessor.kt:159)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl$Task.execute(JupyterExecutorImpl.kt:41)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl.executorThread$lambda$0(JupyterExecutorImpl.kt:83)

    	at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

    

    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled

    at Cell In[53], line 10


然而，出于性能原因，大多数其他流操作符不会自行执行额外的取消检查。例如，如果你使用 `IntRange.asFlow` 扩展来编写相同的繁忙循环并且不挂起任何地方，那么就没有取消检查：


```kotlin
runBlocking<Unit> {
    (1..5).asFlow().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
```

    1
    2
    3
    4
    5



    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#90":BlockingCoroutine{Cancelled}@2fc03ec9

    	at kotlinx.coroutines.JobSupport.cancel(JobSupport.kt:1685)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel(CoroutineScope.kt:309)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel$default(CoroutineScope.kt:307)

    	at Line_55_jupyter$1$1.emit(Line_55.jupyter.kts:3) at Cell In[54], line 3

    	at Line_55_jupyter$1$1.emit(Line_55.jupyter.kts:2) at Cell In[54], line 2

    	at kotlinx.coroutines.flow.FlowKt__BuildersKt$asFlow$$inlined$unsafeFlow$9.collect(SafeCollector.common.kt:111)

    	at Line_55_jupyter$1.invokeSuspend(Line_55.jupyter.kts:2) at Cell In[54], line 2

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:94)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:70)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:48)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at Line_55_jupyter.<init>(Line_55.jupyter.kts:1) at Cell In[54], line 1

    	at java.base/jdk.internal.reflect.DirectConstructorHandleAccessor.newInstance(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstance(Unknown Source)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.evalWithConfigAndOtherScriptsResults(BasicJvmScriptEvaluator.kt:122)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke$suspendImpl(BasicJvmScriptEvaluator.kt:48)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke(BasicJvmScriptEvaluator.kt)

    	at kotlin.script.experimental.jvm.BasicJvmReplEvaluator.eval(BasicJvmReplEvaluator.kt:49)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl$eval$resultWithDiagnostics$1.invokeSuspend(InternalEvaluatorImpl.kt:138)

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl.eval(InternalEvaluatorImpl.kt:138)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute_L4Nmkdk$lambda$9$lambda$1(CellExecutorImpl.kt:80)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withHost(ReplForJupyterImpl.kt:794)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute-L4Nmkdk(CellExecutorImpl.kt:78)

    	at org.jetbrains.kotlinx.jupyter.repl.execution.CellExecutor.execute-L4Nmkdk$default(CellExecutor.kt:14)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evaluateUserCode-wNURfNM(ReplForJupyterImpl.kt:616)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalExImpl(ReplForJupyterImpl.kt:474)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx$lambda$20(ReplForJupyterImpl.kt:467)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withEvalContext(ReplForJupyterImpl.kt:447)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx(ReplForJupyterImpl.kt:466)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6$lambda$5(IdeCompatibleMessageRequestProcessor.kt:160)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedIn(IdeCompatibleMessageRequestProcessor.kt:354)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16$lambda$15(IdeCompatibleMessageRequestProcessor.kt:368)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedErr(IdeCompatibleMessageRequestProcessor.kt:343)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16(IdeCompatibleMessageRequestProcessor.kt:367)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedOut(IdeCompatibleMessageRequestProcessor.kt:335)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO(IdeCompatibleMessageRequestProcessor.kt:366)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6(IdeCompatibleMessageRequestProcessor.kt:159)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl$Task.execute(JupyterExecutorImpl.kt:41)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl.executorThread$lambda$0(JupyterExecutorImpl.kt:83)

    	at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

    

    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled

    at Cell In[54], line 3


## 使繁忙的流可取消

在您有带有协程的繁忙循环的情况下，您必须显式检查取消。您可以添加 `.onEach { currentCoroutineContext().ensureActive() }`，但有一个开箱即用的 `cancellable` 操作符可供使用：


```kotlin
runBlocking<Unit> {
    (1..5).asFlow().cancellable().collect { value ->
        if (value == 3) cancel()
        println(value)
    }
}
```

    1
    2
    3



    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled; job="coroutine#91":BlockingCoroutine{Cancelled}@486df746

    	at kotlinx.coroutines.JobSupport.cancel(JobSupport.kt:1685)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel(CoroutineScope.kt:309)

    	at kotlinx.coroutines.CoroutineScopeKt.cancel$default(CoroutineScope.kt:307)

    	at Line_56_jupyter$1$1.emit(Line_56.jupyter.kts:3) at Cell In[55], line 3

    	at Line_56_jupyter$1$1.emit(Line_56.jupyter.kts:2) at Cell In[55], line 2

    	at kotlinx.coroutines.flow.CancellableFlowImpl$collect$2.emit(Context.kt:278)

    	at kotlinx.coroutines.flow.FlowKt__BuildersKt$asFlow$$inlined$unsafeFlow$9.collect(SafeCollector.common.kt:111)

    	at kotlinx.coroutines.flow.CancellableFlowImpl.collect(Context.kt:276)

    	at Line_56_jupyter$1.invokeSuspend(Line_56.jupyter.kts:2) at Cell In[55], line 2

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:94)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:70)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:48)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at Line_56_jupyter.<init>(Line_56.jupyter.kts:1) at Cell In[55], line 1

    	at java.base/jdk.internal.reflect.DirectConstructorHandleAccessor.newInstance(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source)

    	at java.base/java.lang.reflect.Constructor.newInstance(Unknown Source)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.evalWithConfigAndOtherScriptsResults(BasicJvmScriptEvaluator.kt:122)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke$suspendImpl(BasicJvmScriptEvaluator.kt:48)

    	at kotlin.script.experimental.jvm.BasicJvmScriptEvaluator.invoke(BasicJvmScriptEvaluator.kt)

    	at kotlin.script.experimental.jvm.BasicJvmReplEvaluator.eval(BasicJvmReplEvaluator.kt:49)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl$eval$resultWithDiagnostics$1.invokeSuspend(InternalEvaluatorImpl.kt:138)

    	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)

    	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)

    	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)

    	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)

    	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)

    	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)

    	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.InternalEvaluatorImpl.eval(InternalEvaluatorImpl.kt:138)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute_L4Nmkdk$lambda$9$lambda$1(CellExecutorImpl.kt:80)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withHost(ReplForJupyterImpl.kt:794)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.CellExecutorImpl.execute-L4Nmkdk(CellExecutorImpl.kt:78)

    	at org.jetbrains.kotlinx.jupyter.repl.execution.CellExecutor.execute-L4Nmkdk$default(CellExecutor.kt:14)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evaluateUserCode-wNURfNM(ReplForJupyterImpl.kt:616)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalExImpl(ReplForJupyterImpl.kt:474)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx$lambda$20(ReplForJupyterImpl.kt:467)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.withEvalContext(ReplForJupyterImpl.kt:447)

    	at org.jetbrains.kotlinx.jupyter.repl.impl.ReplForJupyterImpl.evalEx(ReplForJupyterImpl.kt:466)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6$lambda$5(IdeCompatibleMessageRequestProcessor.kt:160)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedIn(IdeCompatibleMessageRequestProcessor.kt:354)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16$lambda$15(IdeCompatibleMessageRequestProcessor.kt:368)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedErr(IdeCompatibleMessageRequestProcessor.kt:343)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO$lambda$16(IdeCompatibleMessageRequestProcessor.kt:367)

    	at org.jetbrains.kotlinx.jupyter.streams.BlockingSubstitutionEngine.withDataSubstitution(SubstitutionEngine.kt:70)

    	at org.jetbrains.kotlinx.jupyter.streams.StreamSubstitutionManager.withSubstitutedStreams(StreamSubstitutionManager.kt:118)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.withForkedOut(IdeCompatibleMessageRequestProcessor.kt:335)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.evalWithIO(IdeCompatibleMessageRequestProcessor.kt:366)

    	at org.jetbrains.kotlinx.jupyter.messaging.IdeCompatibleMessageRequestProcessor.processExecuteRequest$lambda$7$lambda$6(IdeCompatibleMessageRequestProcessor.kt:159)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl$Task.execute(JupyterExecutorImpl.kt:41)

    	at org.jetbrains.kotlinx.jupyter.execution.JupyterExecutorImpl.executorThread$lambda$0(JupyterExecutorImpl.kt:83)

    	at kotlin.concurrent.ThreadsKt$thread$thread$1.run(Thread.kt:30)

    

    kotlinx.coroutines.JobCancellationException: BlockingCoroutine was cancelled

    at Cell In[55], line 3


## 流 (Flow) 与响应式流 (Reactive Streams)

对于熟悉 响应式流 (`Reactive Streams`) 或 `RxJava` 和 `Project Reactor` 等响应式框架的人来说，流 (`Flow`) 的设计可能看起来非常熟悉。

确实，它的设计受到了响应式流及其各种实现的启发。但 `Flow` 的主要目标是拥有尽可能简单的设计，易于与 `Kotlin` 和挂起兼容，并尊重结构化并发。如果没有响应式领域的先驱者及其巨大的工作，实现这一目标将是不可能的。你可以在 Reactive Streams and Kotlin Flows 文章中阅读完整的故事。

虽然概念上有所不同，但 `Flow` 是 一个响应式流，并且可以将其转换为响应式（符合规范和 TCK）的 Publisher，反之亦然。`kotlinx.coroutines` 开箱即用地提供了此类转换器，可以在相应的响应式模块中找到（`kotlinx-coroutines-reactive` 用于 `Reactive Streams`，`kotlinx-coroutines-reactor` 用于 `Project Reactor`，`kotlinx-coroutines-rx2/kotlinx-coroutines-rx3` 用于 `RxJava2/RxJava3`）。集成模块包括与 `Flow` 之间的转换、与 `Reactor` 的 `Context` 集成以及与各种响应式实体协作的挂起友好方式。
