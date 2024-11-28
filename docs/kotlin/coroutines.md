# Kotlin Coroutines

Coroutines behave like Threads but they are not Threads. They can be called lightweight Threads. When a Coroutine sleeps the coroutine will be slept but the Thread will be freed up again. After the sleep the Coroutine continue may in another Thread. You can run millions of Coroutines, but not millions of Threads.

## Coroutine Scope and Context

### Scope

- a coroutine must always be started in a scope
- Cancelling a scope cancels all coroutines running in that scope

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default)
    
    val job = scope.launch {
        delay(1000L)
        println("Task completed!")
    }
    
    delay(500L)
    scope.cancel()
    println("Scope canceled")
}
```

### Context

- define behavior of coroutines
- contains elements (dispatchers, job objects) for configuration of coroutine execution
- is an indexed set of elements where each element in the set has a unique key
- contexts can be combined

Combine a dispatcher + coroutine name:

```kotlin
fun main() = runBlocking {
    val context = Dispatchers.Default + CoroutineName("ExampleCoroutine")
    
    val job = launch(context) {
        println("Coroutine context: $coroutineContext")
        delay(1000L)
        println("Task completed!")
    }
    
    job.join()
}
```

Use a SupervisorJob to make child coroutines independent, so that if one child coroutine fails, it doesn't cause the others to be canceled:

```kotlin
fun main() = runBlocking {
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(coroutineContext + supervisor)

    val job1 = scope.launch {
        delay(500L)
        println("Task 1 completed")
    }

    val job2 = scope.launch {
        delay(1000L)
        throw RuntimeException("Task 2 failed")
    }

    val job3 = scope.launch {
        delay(1500L)
        println("Task 3 completed")
    }

    try {
        joinAll(job1, job2, job3)
    } catch (e: Exception) {
        println("Caught exception: $e")
    }
    supervisor.cancel()
}
```

## Fork/Join equivalent

- use `val x async  { }; x.await()` to launch something async and wait for it

## Launching coroutines

```kotlin
// Using threads
thread {
    sleep(1000)
    println("World")
}

print("Hello, ")
Thread.sleep(1500)

// Using launch
launch {
    delay(1000)             // does not block thread. Only sleeps the coroutine
    // Thread.sleep(1000)   // we can block the main Thread from a coroutine
    println("World")
}

print("Hello, ")
Thread.sleep(1500)          // blocks main thread
```

`launch{}` launches the code async immediately. Instead of `Thread.sleep` we use `delay`.

## Functions and Coroutine Builders

### Launch

- Coroutines have to be run inside a context
- some funcs can only be called inside coroutines (e.g. `delay()`). To let the compiler know that this func can only be called in a coroutine we mark them with `suspend`
- `launch{}` runs the code immediately in a coroutine without waiting for it

### RunBlocking

- `runBlocking{}` runs the code inside but waits for it to finish. Can be used to make the whole main func to run in a coroutine and wait for it
- can be used in tests

```kotlin
suspend fun doWork() {}

class SimpleTest {
    @Test
    fun aTest() = runBlocking {
        dowork()
        assertEquals()
    }
}
```

## Waiting, Join-to, Cancel Coroutines

### Waiting

- Join: calling code blocks until coroutine is finished. `launch{}` returns `Job` which has a `join()` func.

```kotlin
val job = launch {delay(1000); println("World")}
println("Hello")
job.join()
```

### Cancellation

- We must check if a coroutine is cancellable. This has to be checked in a suspending func. This is called cooperative. All built-in funcs are cooperative. Either our suspend func calls a built-in suspend func, or explicitly check if in the job state

```kotlin
val job = launch {
    delay(10)
    println(".")
    // Thread.sleep(1)      <- this would not cancel as it is not cooperative
}
delay(2500)
job.cancel()        // request cancellation
job.join()          // wait for cancellation to happen
println("finally done")
// fast way
job.cancelAndJoin()
```

- we can check the cancellation via `isActive` inside CoroutineScopes
- 1. `yield()` can check in a coroutine if we are cancelled and eventually cancel
- 2. check actively via `if(!isActive)`

In our own code:

```kotlin
// 1
val job = launch {
    delay(10)
    println(".")
    yield()     // checks for cancellation and may cancel
    Thread.sleep(11111)
}
// 2
val job = launch {
    delay(10)
    println(".")
    if(!isActive) throw CancellationException // or: reaturn@launch
    Thread.sleep(11111)
}
```

### Handle Exceptions

- Nice for closing resources: wrap async code in try/catch, on CancellationException close resources. This must be non cancellable.
- we can pass Exceptions to cancel(), but if they are not caught we will kill the app. Never use anything else than `CancellationException` as a reason for `cancel()`

```kotlin
try {
    val job = launch {
        delay(10)
        println(".")
        yield()     // checks for cancellation and may cancel
        Thread.sleep(11111)
    }
} catch(ex: CancellationException) {
    println("Cancelled")
} finally {
    run(NonCancellable) { // now this is not cancellable by another coroutine
        println("Finally")
    }
}
```

### Timeouts

```kotlin
val job = withTimeoutOrNull(100) {
    delay(10)
    println(".")
    yield()     // checks for cancellation and may cancel
    Thread.sleep(11111)
}
if(job == null) {
    println("timedout")
}
delay(1000)
```

## Coroutine Contexts

- all coroutines run as part of context, defined by launcher
- context can flow to child coroutines
- contexts can be joined

### Launch in different contexts

```kotlin
// default
jobs += launch{}
// default
jobs += launch(DefaultDispatcher){}
// not confined -> main thread or where parent was started
jobs+= launch(Unconfined){}
// context of parent (e.g. runBlocking)
jobs += launch(coroutineContext){}
// ForkJoinPool.commonPool
jobs += launch(CommonPool){}
// dedicated thread (expensive operation and thread has to be managed)
jobs += launch(newSingleThreadContext("myThread")){}
```

- `Unconfined` starts in the thread of parent but once a suspending func was called it may continues in another thread
- use `newSingleThreadContext` **always*- with `use{}` so that it is always closed: `newSingleThreadContext("mySTC").use{}`

### Access coroutines

- `coroutineContext` is available in coroutines

```kotlin
val job = launch {
    println("IsActive: ${coroutineContext[job.Key]!!.isActive}")
}
job.join()
```

### Parents - Childs

```kotlin
val outer = launch{
    launch(coroutineContext) { // <---pass context to child
        repeat(1000) {
            print('.')
            delay(1)
        }
    }
}
outer.join() // <--- wait on outer coroutine which waits for the inner coroutine
outer.cancelAndJoin() // <--- cancels both due to relationship
outer.cancelChildren() // <--- cancel only childs
println('finished')
```

- when children are cancelled, exceptions are propagated to parent

### Combine Coroutines

- contexts are maps and can be combined
- keys will be overridden
- missing keys are not added
- order may be important

```kotlin
runBlocking {
    val job = launch(CoroutineName("myCoroutine")+ coroutineContext)
}
```

## Get Data from Coroutines & compose funcs

- launch builder starts coroutine and returns job
- async coroutine builder returns Deferred which can be used later. Deferred inherits from Job

```kotlin
suspend doSth1(): Int {
    delay(100)
    println("working 1")
    return Random(System.currentTimeMillis()).nextInt(42)
}

suspend doSth2(): Int {
    delay(200)
    println("working 2")
    return Random(System.currentTimeMillis()).nextInt(42)
}

fun main(args: Array<String>) {
    val job = launch {
        // start both async, wait for both
        val r1:Deferred<Int> = async{ doSth1() }
        val r2 = async{ doSth2() }
        println("result ${r1.await() + r2.await()}")
    }
    job.join()
}
```

- async jobs run concurrently. If we wait for them we block until this async is done

### Define an async func

```kotlin
fun main(args: Array<String>) {
    val result = doWorkAsync("wasd")
    runBlocking { // we need to setup a coroutine to access the Deferred values
        println(result.await())
    }
}

// not a suspend func, but using coroutine builder
fun doWorkAsync(msg: String): Deferred<Int> = async {
        println(msg)
        delay(100)
        return@async 42
    }
```

### Lazy evaluations

```kotlin
fun main(args: Array<String>) {
    val job = launch {
        val result = async(coroutineContext){ doWorkLazy("wasd") } // define as child, so that main coroutine waits
        println("Result: ${result.await()}")
        val result2 = async(start = CoroutineStart.LAZY) { doWorkLazy("wasd") } // this is only started when await is called!!!!
    }
    job.join()
}

// not a suspend func, but using coroutine builder
suspend fun doWorkLazy(msg: String): Int {
        println(msg)
        delay(100)
        return 42
    }
```

## Channels for in-coroutine-communication

- you can send/receive to/from a channel
- channels block
- can create buffer channels
- need to know when channel has finished

```kotlin
fun main(args: Array<String>) {
    val channel = Channel<Int>()

    val job = launch {
        for(x in 1..5) {
            print("send $x")
            channel.send(x) // channel is then blocked until someone receives the value
        }
        // always close a channel
        channel.close()
    }
    // not so good as we must know how many items will be sent to the channel
    repeat(4) {
        println("receive" ${channel.receive()}) // unblocks channel so that next item can be send; receive blocks as well. If we want to receive sth. but channel is empty the code waits/blocks until sth. is received
    }
    // better approach
    for(y in channel) {
        println("receive" $y)
    }

    job.join()
}
```

### Make code above easier with built-in `ProducerJob`

```kotlin
fun produceNumbers(): ProducerJob<Int> = produce {
    // we are already in a coroutine here
    for(x in 1..5) {
        println("send $x")
        send(x)
    }
    println("done")
}

fun main(args: Array<String>) {
    val channel = produceNumbers()

    channel.consumeEach{
        println(it)
    }
    println("Main done")
}
```

### Pipelining data from one coroutine to another

```kotlin
fun produceNumbers(): ProducerJob<Int> = produce {
    // we are already in a coroutine here
    val x = 1
    while(true) {
        send(x++)
    }
}

fun produceSquareNumbers(numbers: ReceiveChannel<Int>): ProducerJob<Int> = produce {
    // we are already in a coroutine here
    for (x in numbers) {
        send(x*x)
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    val square = produceSquareNumbers(producer)

    for(i in 1..5) {
        println(square.receive())
    }
    square.cancel()
    producer.cancel()
    println("Main done")
}
```

### Multiple consumers / fan-out

```kotlin
fun produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}

fun consumer(id: Int, channel: ReceiveChannel<Int>) = launch {
    channel.consumeEach {
        println("Processor #$id received $it in thread ${Thread.currentThread().name}")
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val producer = produceNumbers()
    repeat(5) { consumer(it, producer) }
    println("launched")
    delay(950)
    producer.cancel() // cancel producer coroutine and thus kill them all
}
```

### Fan-In

```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, interval: Long) {
    while (true) {
        delay(interval)
        channel.send(s)
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<String>()
    launch(coroutineContext) { sendString(channel, "foo", 200L) }
    launch(coroutineContext) { sendString(channel, "BAR!", 500L) }
    repeat(6) { // receive first six
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
}
```

### Buffered Channels (channels which do not block until buffer is full)

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>(4) // create buffered channel
    val sender = launch(coroutineContext) {
        // launch sender coroutine
        repeat(10) {
            println("Sending $it") // print before sending each element
            channel.send(it) // will suspend when buffer is full
        }
    }
    // don't receive anything... just wait....
    delay(1000)
    launch { repeat(10) { println(" --Receiving ${channel.receive()}") } }
    sender.cancel() // cancel sender coroutine
}
```

### Fairness of Channels

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val discusison = Channel<Comment>()

    launch(coroutineContext) { child("he did it", discussion)}
    launch(coroutineContext) { child("she did it", discussion)}

    discussion.send(Comment(0))
    delay(1000)
    coroutineContext.cancel()
}

suspend fun child(text: String, discussion: Channel) {
    for(comment in discussion) {
        comment.count++
        println("$text $comment")
        delay(300)
        discussion.send(comment)
    }
}
```

- when running this code we will see that every coroutine gets the same amount

### Load Balancing Channels - Fan-in and Fan-out

```kotlin
data class Work(var x: Long = 0, var y: Long = 0, var z: Long = 0)

val numberOfWorkers = 10
var totalWork = 20
val finish = Channel<Boolean>()

var workersRunning = AtomicInteger()

suspend fun worker(input: Channel<Work>, output: Channel<Work>) {

    workersRunning.getAndIncrement()
    for (w in input) {
        w.z = w.x - w.y
        delay(w.z)
        output.send(w)
    }
    workersRunning.getAndDecrement()
    if(workersRunning.get() === 0)
    {
        output.close()
        println("Closing output")
    }
}

fun run() {
    val input = Channel<Work>()
    val output = Channel<Work>()

    println("Launch workers")
    repeat (numberOfWorkers) {
        launch { worker(input, output) }
    }
    launch { sendLotsOfWork(input) }
    launch { receiveLotsOfResults(output) }
}

suspend fun receiveLotsOfResults(channel: Channel<Work>) {

    println("receiveLotsOfResults start")

    for(work in channel) {
        println("${work.x}*${work.y} = ${work.z}")
    }
    println("receiveLotsOfResults done")
    finish.send(true)
}

suspend fun sendLotsOfWork(input: Channel<Work>) {
    repeat(totalWork) {
        input.send(Work((0L..100).random(), (0L..10).random()))
    }
    println("close input")
    input.close()
}

fun main(args: Array<String>) {
    run()
    runBlocking { finish.receive() }
    println("main done")
}

private object RandomRangeSingleton : Random()


fun ClosedRange<Long>.random() = (RandomRangeSingleton.nextInt((endInclusive.toInt() + 1) - start.toInt()) + start)
```

## Waiting on multiple coroutines using select

### Simple Select

- allow multiple channels to be awaited on
- select is biased to first channel in list

```kotlin
fun producer1() = produce{
    while(true) {
//        delay(200)
        send("from producer 1")
    }
}

fun producer2() = produce{
    while(true) {
//        delay(300)
        send("from producer 2")
    }
}

suspend fun selector(message1: ReceiveChannel<String>, message2: ReceiveChannel<String>) {
    select<Unit> {
        message1.onReceive { value -> // in case there is a receive here it will always be executed
            println(value)
        }
        message2.onReceive { value ->
            println(value)
        }
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val m1 = producer1()
    val m2 = producer2()

    repeat(15) {
        selector(m1, m2)
    }
}
```

### Select with closed Channels

- use `onReceiveOrNull`

```kotlin
suspend fun selector(message1: ReceiveChannel<String>, message2: ReceiveChannel<String>): String {
    select<String> {
        message1.onReceiveOrnUll { value ->
            value ?: "Channel 1 is closed "
        }
        message2.onReceiveOrnUll { value ->
            value ?: "Channel 2 is closed "
        }
    }
}
```

### Use side Channels

- for non blocking send
- polling is possible
- timeout is possible

```kotlin
fun produceNumbers(side: SendChannel<Int>) = produce<Int> {
    for (num in 1..10) {
        delay(100)
        select<Unit> {      // if I can send to one channel -> send it
            onSend(num){}
            side.onSend(num){}
        }
    }
    println("Done sending")
}

fun main1(args: Array<String>) = runBlocking<Unit> {

    val side = Channel<Int>()

    launch { side.consumeEach { println("side $it") }}

    val producer = produceNumbers(side)

    producer.consumeEach {
        println("$it")
        delay(500)
    }
}
```

- result will be that most is consumed from side channel

### Timeout

```kotlin
fun producer() = produce {
    var i = 0
    while(True) {
        delay(5000)
        send(i++)
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    var msg = producer()
    select<Unit> {
        msg.onReceive {
            println(it)
        }
        onTimeout(4000) {
            println("timed out")
        }
    }
}
```

## Actors

- lightweight processes
- no shared state
- can communicate via messages
- are channels with state
- can protect data
- no state shared -> no locks needed
- 3 parts: coroutine, state, messages

### Actor to increment a counter safely

```kotlin
suspend fun run(context: CoroutineContext, numberOfJobs: Int, count: Int, action: suspend () -> Unit): Long {
    // action is repeated by each coroutine
    return measureTimeMillis {
        val jobs = List(numberOfJobs) {
            launch(context) {
                repeat(count) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
}

sealed class CounterMsg
object InitCounter : CounterMsg()
object IncCounter : CounterMsg()
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg()

fun counterActor() = actor<CounterMsg> {
    var counter = 0
    for(msg in channel) {
        when(msg) {
            is InitCounter -> counter = 0
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = 100
    val count = 10000

    val counter = counterActor()

    counter.send(InitCounter)

    val time = run(CommonPool, jobs, count) {
        counter.send(IncCounter)
    }

    var response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))

    println("Completed ${jobs * count} actions in $time ms")
    println("result is ${response.await()}")
}

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
```
