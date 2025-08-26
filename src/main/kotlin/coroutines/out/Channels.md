# 前置依赖


```kotlin
%useLatestDescriptors
%use coroutines
```

# 通道

Deferred 值提供了一种在协程之间传输单个值的便捷方式。通道 (Channels) 提供了一种传输值流的方式。

## 通道基础

`[通道 (Channel)]` 在概念上与 `BlockingQueue` 非常相似。一个主要区别是，它没有阻塞的 `put` 操作，而是有一个挂起的 `send` 操作；它没有阻塞的 `take` 操作，而是有一个挂起的 `receive` 操作。


```kotlin
import kotlinx.coroutines.channels.Channel

runBlocking {
    val channel = Channel<Int>()
    launch {
        // this might be heavy CPU-consuming computation or async logic,
        // we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

    1
    4
    9
    16
    25
    Done!


## 关闭和迭代通道

与队列不同，通道可以被关闭，以表明不再有元素到来。在接收端，使用常规的 `for` 循环从通道接收元素非常方便。

概念上，`close` 就像是向通道发送一个特殊的关闭令牌。一旦接收到这个关闭令牌，迭代就会停止，因此可以保证在关闭之前所有先前发送的元素都会被接收：


```kotlin
runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}
```

    1
    4
    9
    16
    25
    Done!


## 构建通道生产者

协程生成元素序列的模式非常常见。这是并发代码中常见的_生产者-消费者_模式的一部分。你可以将这样的生产者抽象为一个以通道作为参数的函数，但这与函数必须返回结果的常理相悖。

有一个名为 `produce` 的便捷协程构建器，它使得在生产者端正确实现变得容易，还有一个扩展函数 `consumeEach`，它在消费者端替代了 `for` 循环：


```kotlin
import kotlinx.coroutines.channels.ReceiveChannel
import kotlinx.coroutines.channels.consumeEach
import kotlinx.coroutines.channels.produce

fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

runBlocking {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

    1
    4
    9
    16
    25
    Done!


## 使用管道生成素数

以下示例打印前十个素数，并在主线程的上下文中运行整个管道。由于所有协程都在主 `runBlocking` 协程的作用域内启动，我们无需保留所有已启动协程的显式列表。我们在打印前十个素数后，使用 `cancelChildren` 扩展函数来取消所有子协程。


```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}

runBlocking {
    var cur = numbersFrom(2)
    repeat(10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
}
```

    2
    3
    5
    7
    11
    13
    17
    19
    23
    29


请注意，你也可以使用标准库中的 `iterator` 协程构建器来构建相同的管道。将 `produce` 替换为 `iterator`，`send` 替换为 `yield`，`receive` 替换为 `next`，`ReceiveChannel` 替换为 `Iterator`，并摆脱协程作用域。你也不再需要 `runBlocking`。然而，如上所示使用通道的管道的优点是，如果你在 `Dispatchers.Default` 上下文中运行它，它实际上可以使用多个 CPU 核心。

无论如何，这是一种极其不切实际的寻找素数的方法。在实践中，管道确实涉及其他一些挂起调用（例如对远程服务的异步调用），并且这些管道无法使用 `sequence/iterator` 构建，因为它们不允许任意挂起，这与完全异步的 `produce` 不同。

## 缓冲通道

到目前为止所示的通道都没有缓冲区。无缓冲通道在发送者和接收者相遇（即会合 (rendezvous)）时传输元素。如果 `send` 首先被调用，那么它会挂起直到 `receive` 被调用；如果 `receive` 首先被调用，它会挂起直到 `send` 被调用。

`Channel()` 工厂函数和 `produce` 构建器都接受一个可选的 `capacity` 参数来指定_缓冲区大小_。缓冲区允许发送者在挂起之前发送多个元素，类似于具有指定容量的 `BlockingQueue`，后者在缓冲区满时会阻塞。

请看以下代码的行为：


```kotlin
runBlocking<Unit> {
    val channel = Channel<Int>(4) // create buffered channel
    val sender = launch { // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    // don't receive anything... just wait....
    delay(500)
    sender.cancel() // cancel sender coroutine
}
```

    Sending 0
    Sending 1
    Sending 2
    Sending 3
    Sending 4


前四个元素被添加到缓冲区，发送者在尝试发送第五个时挂起。

## 通道是公平的
通道的 `send` 和 `receive` 操作对于从多个协程调用它们的顺序是_公平的_。它们以先进先出 (`first-in` `first-out`) 顺序服务，例如，第一个调用 `receive` 的协程会获取元素。在以下示例中，两个协程“ping”和“pong”正在从共享的“table”通道接收“ball”对象。


```kotlin
data class Ball(var hits: Int)

runBlocking {
    val table = Channel<Ball>() // a shared table
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    coroutineContext.cancelChildren() // game over, cancel them
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        delay(300) // wait a bit
        table.send(ball) // send the ball back
    }
}
```

    ping Ball(hits=1)
    pong Ball(hits=2)
    ping Ball(hits=3)
    pong Ball(hits=4)


“ping”协程首先启动，所以它是第一个接收到球的。尽管“ping”协程在将球送回“table”后立即再次开始接收球，但球却被“pong”协程接收了，因为它已经在等待它：

## 计时器通道

计时器通道 (`Ticker channel`) 是一种特殊的会合 (`rendezvous`) 通道，它在自上次从该通道消费以来经过给定延迟后，每次都会生产 `Unit`。虽然它单独使用可能看起来无用，但它是创建复杂基于时间的 `produce` 管道以及进行窗口化和其他时间相关处理的运算符的有用构建块。 计时器通道可以在 `select` 中使用，以执行“时钟滴答”操作。

要创建此类通道，请使用工厂方法 `ticker`。要表明不再需要更多元素，请在其上使用 `ReceiveChannel.cancel` 方法。

现在让我们看看它在实践中是如何工作的：


```kotlin
import kotlinx.coroutines.channels.ticker

runBlocking<Unit> {
    val tickerChannel = ticker(delayMillis = 200, initialDelayMillis = 0) // create a ticker channel
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // no initial delay

    nextElement =
        withTimeoutOrNull(100) { tickerChannel.receive() } // all subsequent elements have 200ms delay
    println("Next element is not ready in 100 ms: $nextElement")

    nextElement = withTimeoutOrNull(120) { tickerChannel.receive() }
    println("Next element is ready in 200 ms: $nextElement")

    // Emulate large consumption delays
    println("Consumer pauses for 300ms")
    delay(300)
    // Next element is available immediately
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // Note that the pause between `receive` calls is taken into account and next element arrives faster
    nextElement = withTimeoutOrNull(120) { tickerChannel.receive() }
    println("Next element is ready in 100ms after consumer pause in 300ms: $nextElement")

    tickerChannel.cancel() // indicate that no more elements are needed
}
```

    Initial element is available immediately: kotlin.Unit
    Next element is not ready in 100 ms: null
    Next element is ready in 200 ms: kotlin.Unit
    Consumer pauses for 300ms
    Next element is available immediately after large consumer delay: kotlin.Unit
    Next element is ready in 100ms after consumer pause in 300ms: kotlin.Unit


请注意，`ticker` 知道可能的消费者暂停，并且默认情况下，如果发生暂停，会调整下一个生产元素的延迟，试图保持生产元素的固定速率。

可选地，可以指定一个等于 `TickerMode.FIXED_DELAY` 的 `mode` 参数，以保持元素之间的固定延迟。
