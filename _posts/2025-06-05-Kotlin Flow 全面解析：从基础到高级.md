---
theme: a11y-light
excerpt: 'Kotlin Flow 全面解析：从基础到高级！'
---

## 一个典型的 flow 包含哪些部分？

这是一个非常简单的问题，但是也最为我们所忽略。我们来看一个典型的 flow：

```kotlin
val flowA = flowOf(1, 2, 3) .map { it + 1 }

flow.collect { value ->
    println("Received $value")
}
```

由上面的代码我们可以看到一个 flow，一般包含三个部分构造符，collector，还有操作符（可选）。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/6cb7697a76134d4a85b0e48655d4bf36~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1749707563&x-orig-sign=GjTBDe1usA9f2LtnQmzVsjztOvc%3D)

## 一个 flow 通常来说会经历过哪些过程？

这个问题可以看作是对上个问题的补充，考察的是对 kotlin flow 的基础理解。\
[**android 官方文档**](https://developer.android.com/kotlin/flow?hl=zh-cn#context)详细介绍了每个流程。具体流程为

1.  创建数据流
2.  修改数据流
3.  收集数据流
4.  捕获数据流异常
5.  在不同 Coroutine​Context 中运行

希望大家都能牢记这些流程，他对我们后续其他问题的理解会有帮助。

## 一个 flow 可以有多少个 collector？

kotlin 本身并没有对 collector 数量进行限制，每个 collector 都会收到同样的数据。

## flow 到底是热的还是冷的？

默认使用 builder 构造出来的 flow 是冷的，而 StateFlow，SharedFlow 是热的。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/501f979f2a9f41cb91fdbd1351874409~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1749707563&x-orig-sign=g2hXeK8dQIGz8pQkWO8L5I4vxl4%3D)

## 冷流热流的区别

**冷流 (Cold Flow):**

*   **惰性 (Lazy):**  按需执行。
*   **单播 (Unicast) per collection:**  每个收集器获得独立的执行和数据。
*   **生产者为每个收集器重新执行。**

**热流 (Hot Flow):**

*   **主动 (Active):**  可能在没有收集器的情况下就发射数据。
*   **多播 (Multicast) / 广播 (Broadcast):**  多个收集器共享来自同一生产者的数据。
*   **生产者通常只执行一次（或独立于收集器执行）。**
*   新收集器可能**错过**早期数据，除非 Flow 配置了重播 (replay) 机制。

### 那么 cold flow 与 hot flow 的 collect 执行起来有什么区别吗？

```kotlin
val stateFlow = MutableStateFlow("初始值")

launch { stateFlow.collect { println("收集器1: $it") } }
launch { stateFlow.collect { println("收集器2: $it") } }
launch { stateFlow.collect { println("收集器3: $it") } }
delay(10)
stateFlow.emit("修改后的值")

val flowNormal = flowOf(1,2,3)
launch {
    flowNormal.collect {
        println("flowNormal collector1 $it")

    }
}

launch {
    flowNormal.collect {
        println("flowNormal collector2: $it")
    }
}
```

看下上面代码，猜一猜会输出什么？

好了，展示一下执行结果，是不是有些奇怪，**cold flow 每次收集都是独立的**，这也是冷流热流的主要区别之一。

<details>
  <summary>查看结果</summary>

print console：

*   收集器1: 初始值
*   收集器2: 初始值
*   收集器3: 初始值
*   收集器1: 修改后的值
*   收集器2: 修改后的值
*   收集器3: 修改后的值
*   flowNormal collector1 1
*   flowNormal collector1 2
*   flowNormal collector1 3
*   flowNormal collector2: 1
*   flowNormal collector2: 2
*   flowNormal collector2: 3

</details>

## 热流与冷流 collect 是否阻塞后续代码执行？

相信很多好奇的宝宝已经看到上面的代码都运行在独立的 coroutineScope 之中了，你可能会问 我如果去掉 scope 让他们运行在同一个 coroutineScope 可以不可以呢？下面我们改下代码，咱们看看情况。

```kotlin
stateFlow.collect { println("收集器1: $it") } 
stateFlow.collect { println("收集器2: $it") }
delay(10)
stateFlow.emit("修改后的值")
```

猜一下输出结果会是怎样？结果可能让很多人大吃一惊。

<details>
  <summary>查看结果</summary>
  收集器1: 初始值
</details>

我们可以看到 `stateFlow.collect { println("收集器1: $it") }` 这行代码执行完之后，阻塞了后续的代码执行，也就是说当前 flow 挂起在这里了，当然如果感兴趣的同学可以看看源码实现。最后这个规则一定要记住，热流需要放在单独的 scope 之中运行，不然会阻塞后续的任务执行，我曾经就写过一个相关的 bug, 我在 viewmodel scope 里面执行了两个 collect 但是第二个并没有正确执行。

<details>
  <summary>查看源码</summary>

```kt
  // SharedFlow.kt中的接口
public interface SharedFlow<out T> : Flow<T> {
    public val replayCache: List<T>
}

// SharedFlowImpl.kt中的实现(简化)
internal class SharedFlowImpl<T>(
    private val replay: Int,
    private val extraBufferCapacity: Int,
    onBufferOverflow: BufferOverflow
) : MutableSharedFlow<T> {
    // 收集实现
    override suspend fun collect(collector: FlowCollector<T>) {
        val slot = allocateSlot()
        try {
            if (replay > 0) {
                // 尝试从replay缓存中发射值
                val replaySnapshot = replayCache
                for (value in replaySnapshot) {
                    collector.emit(value)
                }
            }
            // 无限循环，等待新值
            while (true) {
                // 获取新值或挂起等待
                val newValue = awaitValue() 
                collector.emit(newValue)
            }
        } finally {
            freeSlot(slot)
        }
    }
    
    // 挂起等待新值
    private suspend fun awaitValue(): T {
        // 如果没有新值，这里会挂起协程，直到有新值发射
        return suspendCancellableCoroutine { continuation ->
            // 将continuation保存到等待列表中
            // 当有新值时，会恢复这个continuation
        }
    }
}
```

</details>

好了上面讲述了热流 collect 方法执行之后，我们来看一下冷流的 collect，贴上代码：

```kt
val stateFlow = MutableStateFlow("初始值")

val fowNormal = flowOf(1, 2, 3)
flowNormal.collect {
   println("flowNormal collector1 $it")
}

flowNormal.collect {
   println("flowNormal collector2: $it")
 }
```

结果见这里：

<details>
  <summary>查看结果</summary>
  结果  

*   flowNormal collector1 1
*   flowNormal collector1 2
*   flowNormal collector1 3
*   flowNormal collector2: 1
*   flowNormal collector2: 2
*   flowNormal collector2: 3

</details>    

通过上面的执行过程我们可以看到 当构建器代码块不发射任何值并正常结束时，`collect`操作会立即完成并继续执行后续代码，不会阻塞。这是因为`flow`构建器中的代码执行完毕后，收集过程就自然结束了。在这里需要说明的是，虽然 collector 执行完之后不会阻塞后续任务，但是还是建议都运行在单独的 scope 里面，因为 collector 里面可能会执行耗时任务，那么此时阻塞会导致后续代码得不到及时的处理。

<details>
  <summary>查看源码</summary>

```kt
// Flow.kt中flow构建器的简化实现
public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block() // 执行构建器中的代码块
    }
}
```

</details>

## flow 会丢失事件吗？

在上面的例子，我贴过这样的一段代码：

```kt
launch { stateFlow.collect { println("收集器1: $it") } }
launch { stateFlow.collect { println("收集器2: $it") } }
launch { stateFlow.collect { println("收集器3: $it") } }
delay(10)
stateFlow.emit("修改后的值")
```

结果可见上面，相信大家都会有疑惑，如果我删除掉 delay(10) 结果会是怎样呢？

*   收集器1: 修改后的值
*   收集器2: 修改后的值

在这里就不卖关子了，直接展示出结果，stateflow 只展示最新的一条。这是一个大家都熟知的知识点：**StateFlow 会展示最新的值**。那么隐藏在他后面的信息是什么呢？既是热流不依赖 collector 就可以发送，所以热流会丢失事件。那么热流如何能够避免丢失事件呢？

1.  使用 SharedFlow，添加 replay 大小 和 `extraBufferCapacity` 避免溢出
2.  使用 buffer() 转成冷流来处理背压

说到背压下一节我们讲一下 冷流与热流的背压策略的不同。

## 冷流与热流的被压策略

> 背压（Backpressure）是指当数据生产速度快于消费速度时如何处理多余数据的策略。在 Kotlin Flow 中，冷流和热流有不同的背压处理机制。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/f75c3919c6724da793e2f2361991cad5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgQ2FwdGFpblo=:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMzU0NDQ4MTIxNzM4MzkxMSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1749707563&x-orig-sign=2qQ%2FE6o2u3eqPo%2B8Nou2gDtCG%2BU%3D)

## 冷流的背压策略

冷流（如通过 `flow {}`构建器创建的流）默认采用**挂起式背压**：

1.  **默认行为**：当下游消费者处理不及时时，上游生产者会自动挂起，等待消费者准备好
2.  **自然协调**：生产者和消费者之间自然形成速度匹配，不需要额外缓冲区
3.  **可配置性**：可以通过操作符修改默认行为

### 冷流背压操作符

冷流可以使用以下操作符修改背压行为：

```kotlin

// 添加缓冲区，可以避免因下游消费过慢阻塞上游生产
flow.buffer(capacity = 10)

// 指定缓冲区溢出策略
flow.buffer(
    capacity = 10,
    onBufferOverflow = BufferOverflow.DROP_OLDEST // 丢弃最旧的元素
)

// 其他背压相关操作符
flow.conflate()  // 只保留最新值，丢弃中间值
flow.collectLatest { ... }  // 取消处理中的值，开始处理最新值
```

对上面的背压操作符我这里举一个例子，现在假设我们有一个 事件中心用来接收服务端的事件，如果服务端一直发送事件过来，我们客户端可能没法及时处理，会导致 flow 上游挂起，消息堆积，此时我们只有两个措施，要么 drop 要么保存。所以 buffer 操作符就这样诞生了，但是 buffer 操作符也不是万能的，一直有消息过来，buffer 也是会满的，那么此时就需要使用不同策略了，可以选择丢弃最旧的信息等。

## 热流的背压策略

热流（如 `SharedFlow` 和 `StateFlow`）有更复杂的背压处理机制，因为它们可能有多个消费者：

### SharedFlow 背压策略

`SharedFlow`在创建时可以配置背压策略：

```kotlin

val sharedFlow = MutableSharedFlow<Int>(
    replay = 1,  // 重放缓存大小
    extraBufferCapacity = 10,  // 额外缓冲区容量
    onBufferOverflow = BufferOverflow.DROP_OLDEST  // 缓冲区溢出策略
)
```

背压参数说明：

*   **replay**：保留最近发射的N个值，供新订阅者立即消费
*   **extraBufferCapacity**：额外缓冲区大小，当所有消费者处理不及时时使用
*   **onBufferOverflow**：缓冲区满时的策略，可选值：
    *   `SUSPEND` ：挂起发射者（默认）
    *   `DROP_OLDEST`：丢弃最旧的值
    *   `DROP_LATEST`：丢弃最新的值

### StateFlow 背压策略

`StateFlow`是一种特殊的 `SharedFlow`，它：始终有 `replay = 1`（只保留最新值）

没有额外缓冲区（` extraBufferCapacity = 0`）
使用 DROP\_OLDEST 溢出策略（总是保留最新值）这意味着 `StateFlow`永远不会因背压而挂起发射者，它总是保留最新值并丢弃旧值。

## 冷流与热流背压策略的关键区别

1.  **默认行为**：
    *   冷流：默认挂起发射者等待消费者
    *   热流：可配置（`SharedFlow`）或固定策略（`StateFlow`）

2.  **多消费者场景**：
    *   冷流：每个消费者获得独立的数据流，背压独立处理
    *   热流：所有消费者共享同一数据流，最慢的消费者可能影响所有人

3.  **缓冲区配置**：
    *   冷流：通过操作符动态添加
    *   热流：在创建时静态配置

4.  **数据丢失风险**：
    *   冷流：默认不会丢失数据
    *   热流：可能配置为丢弃数据（` DROP_OLDEST`/`DROP_LATEST`）

## 实际应用示例

### 冷流背压示例

```kotlin

val slowConsumer = flow {
    for (i in 1..100) {
        delay(10)  // 生产速度快
        emit(i)
    }
}.buffer(10)  // 添加缓冲区
 .flowOn(Dispatchers.Default)  // 在不同协程上下文中执行

// 消费者处理慢
slowConsumer.collect { value ->
    delay(100)  // 消费速度慢
    println("Processed: $value")
}
```

### 热流背压示例

```kotlin
// 配置不同背压策略的SharedFlow
val suspendingFlow = MutableSharedFlow<Int>(
    replay = 0,
    extraBufferCapacity = 0,  // 没有额外缓冲区，会挂起
    onBufferOverflow = BufferOverflow.SUSPEND
)

val droppingFlow = MutableSharedFlow<Int>(
    replay = 0,
    extraBufferCapacity = 10,  // 有缓冲区
    onBufferOverflow = BufferOverflow.DROP_OLDEST  // 缓冲区满时丢弃旧值
)

// StateFlow总是使用DROP_OLDEST策略
val stateFlow = MutableStateFlow(0)
```

一般来说选择合适的背压策略取决于您的应用场景：

*   需要处理所有数据时，使用默认挂起策略
*   只关心最新数据时，使用丢弃策略
*   需要平衡吞吐量和内存使用时，配置适当大小的缓冲区

## Kotlin Flow 与线程安全

#### 冷流的线程安全特性：

*   **独立执行**：每个收集器获得独立的流实例，不同收集器之间不会相互干扰
*   **单线程收集**：默认情况下，单个收集操作是在单一线程上顺序执行的
*   **线程安全**：由于每次收集都是独立的，冷流本身不存在线程安全问题

#### 热流（Hot Flow）

热流的线程安全特性：

**线程安全实现**：内部使用原子操作和同步机制确保线程安全

*   **多线程访问**：支持从多个线程同时发射和收集值
*   **原子性保证**：eimit 和 tryEmit 操作是原子的，确保值的完整性

### Flow 操作符的线程安全性

Flow 操作符（如 `map`、`filter`、`transform`等）的线程安全性取决于：

*   **操作符实现**：大多数操作符保证内部线程安全
*   **用户提供的转换函数**：如果您的转换函数访问共享状态，需要自行确保线程安全
*   **执行上下文**：操作符在哪个调度器上执行会影响线程安全需求

```kotlin
// 这个转换函数访问共享状态，需要确保线程安全
var counter = 0
val flow = flowOf(1, 2, 3)
    .map { 
        counter++ // 注意：这是非线程安全的操作, 当前修改与 counter 并不在同一线程内
        it * 2 
    }
    .flowOn(Dispatchers.Default)
```

#### 线程安全的最佳实践

1.  **避免可变共享状态**：尽量避免在 Flow 操作中访问可变共享状态
2.  **使用不可变数据**：优先使用不可变数据结构，避免并发修改问题
3.  **正确使用调度器**：了解  `flowOn `、 `launchIn`和 `collect`的上下文影响
4.  **注意 StateFlow 更新**：对于 ` StateFlow`，使用`update`函数进行原子更新
5.  **使用线程安全的集合**：当需要在 Flow 中使用集合时，考虑使用线程安全的集合类

#### 常见线程安全陷阱

1.  **收集者上下文**：默认情况下，Flow 在收集者的上下文中执行，可能导致 UI 线程阻塞

```kt
// 错误：可能在主线程上执行耗时操作
lifecycleScope.launch {
    flow.collect { /* 在主线程执行 */ }
}

// 正确：使用适当的调度器
lifecycleScope.launch {
    flow.flowOn(Dispatchers.IO).collect { /* 处理数据 */ }
}
```

2.  **共享可变状态**：在 Flow 操作符中修改共享状态可能导致竞态条件

```kt
// 错误：非线程安全的状态修改
val list = mutableListOf<Int>()
flow.onEach { list.add(it) } // 可能导致竞态条件
// 正确：使用线程安全的收集方式
val result = flow.toList() // 收集到线程安全的集合
```

## The end

受限于文章长度吗，本文只讲述了一些日常开发容易被大家忽略的 kotlin flow 知识点，希望这篇文章能够帮助到你。
