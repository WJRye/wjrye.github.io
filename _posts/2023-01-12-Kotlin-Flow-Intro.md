---
layout: post
title: Kotlin Flow 探索
categories: [Kotlin]
description: Kotlin 协程 和 Flow，它们结合在一起也实现了 响应式编程。
keywords: Kotlin, Flow
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
topmost: false
---

## 响应式编程

因为 Kotlin Flow 是基于 响应式编程 的实现，所以先了解一下 响应式编程 的概念。

首先看下百度百科解释：

> 响应式编程是一种面向数据流和变化传播的编程范式。这意味着可以在编程语言中很方便地表达静态或动态的数据流，而相关的计算模型会自动将变化的值通过数据流进行传播。

这个释义很抽象，难以理解。只知道它的核心是：**数据流**。

如何理解这个数据流，先看下响应式编程 [ReactiveX](https://github.com/ReactiveX) 下的一个框架 [RxJava](https://reactivex.io/intro.html) 。

[RxJava](https://reactivex.io/intro.html) 是基于响应式编程的实现，它的定义：

> RxJava 是 Reactive Extensions 的 Java VM 实现：一个通过使用可观察序列来组合异步和基于事件的程序的库。
> 它扩展了观察者模式以支持数据/事件序列，并添加了运算符，允许您以声明方式组合序列，同时消除了对低级线程、同步、线程安全和并发数据结构等问题的担忧。

看完这个定义，脑袋中也很模糊。下面从 RxJava 应用的一个简单例子来分析：

```
   Observable.just(bitmap).map { bmp->
            //在子线程中执行耗时操作，存储 bitmap 到本地
            saveBitmap(bmp)
        }.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe { bitmapLocalPath ->
                //在主线程中处理存储 bitmap 后的本地路径地址
                refreshImageView(bitmapLocalPath)
        }
```

上面例子中： 将一个 bitmap 存储到本地并返回本地路径，从源数据 bitmap &rarr; 存储 btimap 到本地操作  &rarr;  获取本地图片路径值刷新UI。其实，就可以把这整个过程中按时间发生的事件序列理解为**数据流**。

**数据流**包含提供方（生产者），中介（中间操作），使用方（消费者）：

- 提供方（生产者）：源数据，将数据添加到数据流中；
- 中介（中间操作）：可以修改发送到数据流的值，或修正数据流本身；
- 使用方（消费者）：结果数据，使用数据流中的值。

那么，上面例子中的数据流是：

- 提供方（生产者）：源数据 bitmap；
- 中介（中间操作）：map 操作，存储 btimap 到本地；
- 使用方（消费者）：本地图片路径。

再看下 RxJava 中的数据流解释：

 RxJava 中的数据流由源、零个或多个中间步骤组成，然后是数据消费者或组合器步骤（其中该步骤负责通过某种方式消费数据流）：

 `source.operator1().operator2().operator3().subscribe(consumer);`
 `source.flatMap(value -> source.operator1().operator2().operator3());`

在这里，如果我们想象自己在操作符 operator2 上，向左看 source 被称为**上游**。向右看 subscriber/consumer 称为**下游**。当每个元素都写在单独的行上时，这一点通常更为明显：

```
source
  .operator1()
  .operator2()
  .operator3()
  .subscribe(consumer)
```

这也是 RxJava 的上游、下游概念。

其实，Flow 数据流中参看 RxJava，也可以有这样类似的上游和下游概念：

```
flow
  .operator1()
  .operator2()
  .operator3()
  .collect(consumer)
```

了解了 响应式编程 的核心 **数据流** 后，对 响应式编程 有了初步印象。但是 响应式编程 的实现远不止如此，它还涉及**观察者模式**，**线程调度**等。不管原理这些，用它来做开发有什么好处呢？其实，它主要优点是：

- 对于并发编程，线程切换，没有 callback hell，简化了异步执行的代码；
- 代码优雅，简洁，易阅读和维护。

下面看两个业务例子：

```
     Observable.just(bitmap).map { bmp ->
            //在子线程中执行耗时操作，存储 bitmap 到本地
            saveBitmap(bmp)
        }.map { path ->
            //在子线程中执行耗时操作，上传图片到服务端
            uploadBitmap(path)
        }.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
            .subscribe { downloadUrl ->
                //在主线程中处理获取图片下载地址
            }
```

```
        //从服务端批量下载文件
        Observable.from(downloadUrls).flatMap { downloadUrl ->
            //下载单个文件，返回本地文件
            Observable.just(downloadUrl).map {url-> downloadResource(url) }
        }.map { file ->
            //对文件解压
            unzipFile(file)
        }.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
            .subscribe { folderPath ->
                //拿到文件夹路径
            }
```

所以 响应式编程 的实现，主要是帮我们解决了 并发编程问题，能用优雅简洁的代码做异步事件处理。

Kotlin 协程 和 Flow，它们结合在一起也实现了 响应式编程。在 Kotlin 环境中，再结合 Android 提供 Lifecycle, ViewModel, Flow 的扩展，能让我们在 Android 中做并发编程，异步事件管理如鱼得水。

## Kotlin Flow

Kotlin Flow 就是 Kotlin 数据流，它基于 Kotlin 协程构建。上一篇 [Kotlin 协程探索]({{ site.url }}/2023/01/04/Kotlin-Coroutine-Intro/) 分析了  协程 的大致原理，知道协程就是 Kotlin 提供的一套线程 API 框架，方便做并发编程。那么 Kotlin 协程 和 Flow （数据流）的结合，和 RxJava 框架就有异曲同工之妙。

下面使用 Kotlin 协程 和 Flow 来实现上面 RxJava 的两个业务例子：

```
        GlobalScope.launch(Dispatchers.Main) {
            flowOf(bitmap).map { bmp ->
                //在子线程中执行耗时操作，存储 bitmap 到本地
                Log.d("TestFlow", "saveBitmap: ${Thread.currentThread()}")
                saveBitmap(bmp)
            }.flowOn(Dispatchers.IO).collect { bitmapLocalPath ->
                //在主线程中处理存储 bitmap 后的本地路径地址
                Log.d("TestFlow", "bitmapLocalPath=$bitmapLocalPath: ${Thread.currentThread()}")
            }
        }
        //从服务端批量下载文件
        GlobalScope.launch(Dispatchers.Main) {
            downloadUrls.asFlow().flatMapConcat { downloadUrl ->
                //下载单个文件，返回本地文件
                flowOf(downloadUrl).map { url ->
                    Log.d("TestFlow", "downloadResource:url=$url: ${Thread.currentThread()}")
                    downloadResource(url)
                }
            }.map { file ->
                //对文件解压
                Log.d("TestFlow", "unzipFile:file=${file.path}: ${Thread.currentThread()}")
                unzipFile(file)
            }.flowOn(Dispatchers.IO).collect { folderPath ->
                //拿到文件夹路径
                Log.d("TestFlow", "folderPath=$folderPath: ${Thread.currentThread()}")
            }
        }
控制台结果输出：
TestFlow: saveBitmap: Thread[DefaultDispatcher-worker-1,5,main]
TestFlow: bitmapLocalPath=/mnt/sdcard/Android/data/com.wangjiang.example/files/images/flow.png: Thread[main,5,main]

TestFlow: downloadResource:url=https://www.wangjiang.example/coroutine.zip: Thread[DefaultDispatcher-worker-1,5,main]
TestFlow: unzipFile:file=/mnt/sdcard/Android/data/com.wangjiang.example/files/zips/coroutine.zip: Thread[DefaultDispatcher-worker-1,5,main]
TestFlow: downloadResource:url=https://www.wangjiang.example/flow.zip: Thread[DefaultDispatcher-worker-1,5,main]
TestFlow: unzipFile:file=/mnt/sdcard/Android/data/com.wangjiang.example/files/zips/flow.zip: Thread[DefaultDispatcher-worker-1,5,main]
TestFlow: folderPath=/mnt/sdcard/Android/data/com.wangjiang.example/files/zips/coroutine: Thread[main,5,main]
TestFlow: folderPath=/mnt/sdcard/Android/data/com.wangjiang.example/files/zips/flow: Thread[main,5,main]
```

可以看到，和 RxJava 实现的效果是一致的。首先，使用`launch`启动一个协程，然后使用源数据创建一个 `Flow`（数据生产），再经过 `flatMapConcat`, `map` 变换（多个中间操作），最后通过`collect`获取结果数据（数据消费），这其中还包括线程切换：在主线程中启动子线程执行耗时任务，并将耗时任务结果返回给主线程（flowOn 指定了中间操作在 IO 线程中执行）。所以 协程 和 Flow（数据流） 结合，就是 响应式编程 的实现，这对我们来说，使用它可以在 Kotlin 环境中写出优雅的异步代码来做并发编程。

下面再分别来熟悉一下 协程 和 Flow。

### 协程概念

首先来看一下协程中的一些概念和 API。

#### CoroutineScope: 定义协程的 scope。

> CoroutineScope 会跟踪它使用 launch 或 async 创建的所有协程。您可以随时调用 scope.cancel() 以取消正在进行的工作（即正在运行的协程）。在 Android 中，某些 KTX 库为某些生命周期类提供自己的 CoroutineScope。例如，ViewModel 有 viewModelScope，Lifecycle 有 lifecycleScope。不过，与调度程序不同，CoroutineScope 不运行协程。

Kotlin 提供了为 UI 组件使用的 `MainScope`：

```
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```

为应用程序整个生命周期使用的 `GlobalScope`:

```
public object GlobalScope : CoroutineScope {
    /**
     * Returns [EmptyCoroutineContext].
     */
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

因为是应用程序整个生命周期，所以要慎重使用。

也可以自定义 Scope：

```
val scope = CoroutineScope(Job() + Dispatchers.Main)
```

另外，Android KTX 库针对 `CoroutineScope` 做了扩展，所以在 Android 中通常会使用 Activity 或 Fragment 生命周期相关的 `lifecycleScope`，和 ViewModel 生命周期相关的`viewModelScope` 。

```
public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = mInternalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (mInternalScopeRef.compareAndSet(null, newScope)) {
                newScope.register()
                return newScope
            }
        }
    }
```

```
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }

internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```

#### 启动协程: launch 和 async

启动协程有两种方式：

- `launch` ：启动一个新的协程，并返回一个 `Job`，这个 `Job` 是可以取消的 `Job.cancel`；
- `async` ：也会启动一个新的协程，并返回一个 `Deferred` 接口实现，这个接口其实也继承了`Job` 接口，可以使用 `await` 挂起函数等待返回结果。

#### CoroutineContext: 协程上下文

```
val scope = CoroutineScope(Job() + Dispatchers.Main)
```

在 CoroutineScope 中定义了 plus 操作：

```
public operator fun CoroutineScope.plus(context: CoroutineContext): CoroutineScope =
    ContextScope(coroutineContext + context)
```

因为 `Job` 和 `Dispatchers` 顶层都继承了接口 `Element`，而 `Element` 又继承了接口 `CoroutineContext`: 

```
public interface Element : CoroutineContext
```

所以 Job() 和 Dispatchers.Main 可以相加。这里 CoroutineScope 的构造方法中是必须要有 `Job()`，如果没有，它自己也会创建一个 `Job()`：

```
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())
```

Job 和 CoroutineDispatcher 在 `CoroutineContext` 中的作用是：

> Job：控制协程的生命周期。
> CoroutineDispatcher：将工作分派到适当的线程。

#### CoroutineDispatcher：协程调度器与线程

- Dispatchers.Default：默认调度器，指示此协程应在为 cpu 计算操作预留的线程上执行；
- Dispatchers.Main：指示此协程应在为 UI 操作预留的主线程上执行；
- Dispatchers.IO：指示此协程应在为 I/O 操作预留的线程上执行。

```
GlobalScope.launch(Dispatchers.Main) {
}
```

```
withContext(Dispatchers.IO){
}
```

```
.flowOn(Dispatchers.IO)
```

#### 小结

要使用协程，首先创建一个 `scope: CoroutineScope` 来负责管理协程，定义`scope` 时需要指定控制协程的生命周期的  `Job`和将工作分派到适当线程的`CoroutineDispatcher`。定义好 scope 后， 可通过 `scope.launch`启动一个协程，也可以多次使用`scope.launch`启动多个协程，启动的协程可通过 `scope.cancel`取消，但它取消的是 scope 启动的所有协程。如果要取消单个协程，需要使用`scope.launch`返回的 `Job` 来取消`Job.cancel`，这个 Job 控制着单个协程的生命周期。当启动协程后，主线程中的任务依然可以继续执行，在执行`launch{}`时，可以通过 `withContext(Dispatchers.IO)`将协程的执行操作移至一个 I/O 子线程，子线程执行完任务，再将结果返回主线程继续执行。

简单示例：

```
    //主线程分派任务
    private val scope = CoroutineScope(Job() + Dispatchers.Main)

    //管理对应的协程的生命周期
    private var job1: Job? = null

    fun exec() {
        //启动一个协程
        job1 = scope.launch {
            //子线程执行耗时任务
            withContext(Dispatchers.IO){

            }
        }
        //启动一个协程
        val job2 = scope.launch {
            //启动一个协程
            val taskResult1 = async {
                //子线程执行耗时任务
                withContext(Dispatchers.IO){

                }
            }
            val taskResult2 = async {
                //子线程执行耗时任务
                withContext(Dispatchers.IO){

                }
            }
            //taskResult1 和 taskResult2 都返回结果才会继续执行
            taskResult1.await() + taskResult2.await()
        }
    }

    fun cancelJob() {
        //取消 job1 对应的协程
        job1?.cancel("cancel job1")
    }

    fun cancelScope() {
        //取消 scope 对应的所有协程
        scope.cancel("cancel scope")
    }
```

在上面的例子中：

- `scope`：定义主线程分派任务的 scope 来跟踪它使用 launch 或 async 创建的所有协程；
- `job1`：管理它对应的协程的生命周期；
- `withContext(Dispatchers.IO)`：切换到子线程执行耗时任务；
- `cancelJob` 会取消 job1 对应的协程；
- `cancelScope` 会取消 scope 启动的所有协程。

### Flow 数据流

了解了 Kotlin 协程的一些基础 概念和 API 后，知道了协程的基本使用。接下来，再了解一下 Kotlin Flow 相关的概念和 API。

> Kotlin 中的 Flow API 旨在异步处理按顺序执行的数据流。Flow 本质上是一个 Sequence。我们可以像对 Kotlin 中 Sequence 一样来操作Flow：变换，过滤，映射等。**Kotlin Sequences 和 Flow 的主要区别在于 Flow 可以挂起**。

如果有理解 Kotlin Sequence，那其实很好理解 Kotlin Flow。刚好，在前面一篇 [Kotlin 惰性集合操作-序列 Sequence]({{ site.url }}/2023/01/03/Kotlin-Sequence-Intro/)文章中，有分析 Sequence 的原理，这里也可以把 Flow 按照类似的原理进行理解。

```
val sequenceResult = intArrayOf(1, 2, 3).asSequence().map { it * it }.toList()

 MainScope().launch{
            val flowResult = intArrayOf(1, 2, 3).asFlow().map { it * it }.toList(mutableListOf())
        }
```

上面 sequenceResult 和 flowResult 的值都是：`[1, 4, 9]`。

在 Sequence 中，如果没有末端操作，中间操作不会被执行。在 Flow 中也是一样，如果数据流没有数据消费`collect` ，中间操作也不会被执行。

```
flowOf(bitmap).map { bmp ->
                //在子线程中执行耗时操作，存储 bitmap 到本地
                saveBitmap(bmp)
            }.flowOn(Dispatchers.Default)
```

上面代码中，`map` 操作不会被执行。

一个完整的数据流应该包含：数据生产（ `flowOf` , `asFlow`, `flow{}` ）&rarr; 中间操作（`map`, `filter`等）&rarr; 数据消费（`collect`,`asList`,`asSet`等）。下面将分别了解相关操作。

#### 数据流：数据生产

**数据生产主要是通过数据源构建数据流**。可以使用 `Builders.kt` 中提供的 Flow 相关扩展方法，如：

```
intArrayOf(1, 2, 3).asFlow().map { it * it }
```

```
val downloadUrl = "https://github.com/ReactiveX/RxJava"
flowOf(downloadUrl).map { downloadZip(it) }
```

```
(1..10).asFlow().filter { it % 2 == 0 }
```

通常使用 `flowOf` 和 `asFlow`方法直接构建数据流。它们创建的都是冷流：

> **冷流：这段 flow 构建器中的代码直到流被收集（collect）的时候才运行。**

也可以通过 `flow{}` 来构建数据流，使用`emit`方法将数据源添加到数据流中：

```
    flow<Int> {
            emit(1)
            withContext(Dispatchers.IO){
                emit(2)
            }
            emit(3)
        }.map { it * it }
```

不管是 `flowOf`，`asFlow` 还是  `flow{}`，它们都会实现接口 `FlowCollector`：

```
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

internal inline fun <T> unsafeFlow(@BuilderInference crossinline block: suspend FlowCollector<T>.() -> Unit): Flow<T> {
    return object : Flow<T> {
        override suspend fun collect(collector: FlowCollector<T>) {
            collector.block()
        }
    }
}
```

接口 `FlowCollector` 提供的 `emit` 方法，负责将源数据添加到数据流中：

```
public fun interface FlowCollector<in T> {

    /**
     * Collects the value emitted by the upstream.
     * This method is not thread-safe and should not be invoked concurrently.
     */
    public suspend fun emit(value: T)
}
```

**总结**：构建数据流可以使用 Flow 相关扩展方法： `flowOf` , `asFlow`, `flow{}`，它们都是通过接口 `FlowCollector` 提供的 `emit` 方法，将源数据添加到数据流中。

#### 数据流：中间操作

**中间操作主要修改发送到数据流的值，或修正数据流本身**。如 `filter`, `map`, `flatMapConcat` 操作等：

```
intArrayOf(1, 2, 3).asFlow().map { it * it }.collect{ }

(1..100).asFlow().filter { it % 2 == 0 }.collect{ }

val data = hashMapOf<String, List<String>>(
                "Java" to arrayListOf<String>("xiaowang", "xiaoli"),
                "Kotlin" to arrayListOf<String>("xiaozhang", "xiaozhao")
            )
flow<Map<String, List<String>>> {
                emit(data)
            }.flatMapConcat {
                it.values.asFlow()
            }.collect{ }
```

中间操作符有很多，根据使用场景大概可分为：

- 转换操作符：简单转换可以使用过滤 `filter`，映射 `map`操作，复杂转换可以使用变换 `transform`操作；
- 限长过渡操作符：在流触及相应限制的时候会将它的执行取消，可以使用获取 `take`操作，`take(2)`表示只获取前两个值；
- 丢弃操作符：丢弃流中结果值，可以使用丢弃 `drop` 操作，`drop(2)`表示丢弃前两个值；
- 展平操作符：将给定的流展平为单个流，`flatMapConcat` 与 `flattenConcat` 操作表示顺序收集传入的流操作，`flatMapMerge`与`flattenMerge`操作表示并发收集所有传入的流，并将它们的值合并到一个单独的流，以便尽快的发射值操作，`flatMapLatest` 操作表示以展平的方式收集最新的流操作；
- 组合操作符：将多个流组合，`zip`操作表示组合两个流的值，两个流都有值才进行组合操作，`combine` 操作表示组合两个流最新的值，每次组合的时候都是使用每个流最新的值；
- 缓冲操作符：当数据生产比数据消费快的时候，可以使用缓冲`buffer`操作，在数据消费的时候可以缩短时间；
- 合并操作符：合并发射项，不对每个值进行处理，可以使用合并`conflate `操作，跳过中间值；
- flowOn 操作符：更改流发射的上下文，会将 `flowOn`操作前的操作切换到 flowOn 指定的上下文`Dispatchers.Default`，`Dispatchers.IO`，`Dispatchers.Main`，也就是指定前面的操作所执行的线程；

上面介绍了主要的操作符的大致使用场景，操作符详细解释可以查看官方文档：[异步流](https://www.kotlincn.net/docs/reference/coroutines/flow.html#%E5%BC%82%E6%AD%A5%E6%B5%81)。

中间操作符代码示例：

```
(1..3).asFlow().take(2).collect{
                //收集到结果值 1，2
            }
```

```
(1..3).asFlow().drop(2).collect{
                //收集到结果值 3
            }
```

```
    private fun downloadVideo(videoUrl: String): Pair<String, String> {
        return Pair(videoUrl, "videoFile")
    }

    private fun downloadAudio(audioUrl: String): Pair<String, String> {
        return Pair(audioUrl, "audioFile")
    }

    private fun downloadImage(imageUrl: String): Pair<String, String> {
        return Pair(imageUrl, "imageFile")
    }

  MainScope().launch {
            val imageDownloadUrls = arrayListOf<String>("image1", "image2")
            val audioDownloadUrls = arrayListOf<String>("audio1", "audio2", "audio3")
            val videoDownloadUrls = arrayListOf<String>("video1", "video2", "video3", "video4")
            val imageFlows = imageDownloadUrls.asFlow().map {
                downloadImage(it)
            }
            val audioFlows = audioDownloadUrls.asFlow().map {
                downloadAudio(it)
            }
            val videoFlows = videoDownloadUrls.asFlow().map {
                downloadVideo(it)
            }
            merge(imageFlows, audioFlows, videoFlows).flowOn(Dispatchers.IO).onEach {
                Log.d("TestFlow", "result=$it")
            }.collect()
        }
控制台输出结果：
TestFlow: result=(image1, imageFile)
TestFlow: result=(image2, imageFile)
TestFlow: result=(audio1, audioFile)
TestFlow: result=(audio2, audioFile)
TestFlow: result=(audio3, audioFile)
TestFlow: result=(video1, videoFile)
TestFlow: result=(video2, videoFile)
TestFlow: result=(video3, videoFile)
TestFlow: result=(video4, videoFile)

merge 操作符将多个流合并到一个流，支持并发。类似 RxJava 的 zip 操作
```

```
(1..3).asFlow().onStart {
                Log.d("TestFlow", "onStart:${Thread.currentThread()}")
            }.flowOn(Dispatchers.Main).map {
                Log.d("TestFlow", "map:$it,${Thread.currentThread()}")
                if (it % 2 == 0)
                    throw IllegalArgumentException("fatal args:$it")
                it * it
            }.catch {
                Log.d("TestFlow", "catch:${Thread.currentThread()}")
                emit(-1)
            }.flowOn(Dispatchers.IO)
                .onCompletion { Log.d("TestFlow", "onCompletion:${Thread.currentThread()}") }
                .onEach {
                    Log.d("TestFlow", "onEach:$it,${Thread.currentThread()}")
                }.collect()
控制台输出结果：
TestFlow: onStart:Thread[main,5,main]
TestFlow: map:1,Thread[DefaultDispatcher-worker-3,5,main]
TestFlow: map:2,Thread[DefaultDispatcher-worker-3,5,main]
TestFlow: catch:Thread[DefaultDispatcher-worker-3,5,main]
TestFlow: onEach:1,Thread[main,5,main]
TestFlow: onEach:-1,Thread[main,5,main]
TestFlow: onCompletion:Thread[main,5,main]

flowOn 指定 onStart 在主线程中执行（Dispatchers.Main），指定 map 和 catch 在 IO 线程中执行（Dispatchers.IO）
```

**总结**：中间操作其实就是数据流的变换操作，与 Sequence 和 RxJava 的变换操作类似。

#### 数据流：数据消费

**数据消费就是使用数据流的结果值**。末端操作符最常使用 `collect`来收集流结果值：

```
 (1..3).asFlow().collect{
                //收集到结果值 1，2，3
            }
```

除了 `collect` 操作符外，还有一些操作符可以获取数据流结果值：

- `collectLatest`：使用数据流的最新值；
- `toList` 或 `toSet`等：将数据流结果值转换为集合；
- `first`：获取数据流的第一个结果值；
- `single`：确保流发射单个（single）值；
- `reduce` ：累积数据流中的值；
- `fold` ：给定一个初始值，再累积数据流中的值。

末端操作符代码示例：

```
  (1..3).asFlow().collectLatest {
                delay(300)
                //只能获取到3
            }
```

```
//转换为 List 集合 [1,2,3]
 val list = (1..3).asFlow().toList()
 //转换为 Set 集合 [1,2,3]
 val set = (1..3).asFlow().toSet()
```

```
val first = (1..3).asFlow().first()
//first 为第一个结果值 1
```

```
val single = (1..3).asFlow().single()
//流不是发射的单个值，会抛异常
```

```
val reduce = (1..3).asFlow().reduce { a, b ->
                a + b
            }
//reduce 的值为6=1+2+3            
```

```
val fold = (1..3).asFlow().fold(10) { a, b ->
                a + b
            }
 //fold 的值为16=10+1+2+3           
```

除了上面这些末端操作符，在末端之前还关联着一些操作符：

- `onStart`：在数据流结果值收集之前调用；
- `onCompletion`：在数据流结果值收集之后调用；
- `onEmpty`：在数据流完成而不发出任何元素时调用；
- `onEach`：在数据流结果值收集时迭代流的每个值；
- `catch` ：在收集数据流结果时，声明式捕获异常。

末端关联操作符代码示例：

```
 (1..3).asFlow().onStart {
                Log.d("TestFlow", "onStart")
            }.map {
                if (it % 2 == 0)
                    throw IllegalArgumentException("fatal args:$it")
                it * it
            }.catch { emit(-1) }.onCompletion { Log.d("TestFlow", "onCompletion") }.onEach {
                Log.d("TestFlow", "onEach:$it")
            }.collect()

控制台输出结果：
TestFlow: onStart
TestFlow: onEach:1
TestFlow: onEach:-1
TestFlow: onCompletion
```

**总结**：数据流进行数据消费时，可以结合末端操作符输出集合，累积值等，当要监听数据流收集结果值开始或结束，可以使用 `onStart` 和 `onCompletion`，当遇到流抛出异常，可以声明 `catch`进行异常处理。

## 总结

响应式编程，可以理解为一种面向数据流编程的方式，也就是使用数据源构建数据流 &rarr; 修改数据流中的值 &rarr; 处理数据流结果值，在这个过程中，一系列的事件或操作都是按顺序发生的。在 Java 环境中，RxJava 框架实现了响应式编程，它结合了数据流、观察者模式、线程框架；在 Kotlin 环境中，Kotlin 协程和 Flow 结合在一起实现了响应式编程，其中协程就是线程框架，Flow 就是数据流。不管是 RxJava 还是 Kotlin 协程和 Flow 的实现的响应式编程，它们的目的都是为了：使用优雅，简洁，易阅读，易维护的代码来编写并发编程，处理异步操作事件。另外，Android LifeCycle 和 ViewModel 对 Kotlin 协程和 Flow 进行了扩展支持，这也对异步事件进行生命周期管理更方便。

参考文档：

- GitHub RxJava：[RxJava](https://github.com/ReactiveX/RxJava)
- Kotlin Flow 操作符：[异步流](https://www.kotlincn.net/docs/reference/coroutines/flow.html)
- Google Android developer Kotlin Flow ：[Android 上的 Kotlin 数据流](https://developer.android.google.cn/kotlin/flow?hl=zh_cn)

下一篇将探索 Kotlin Flow 冷流和热流。
