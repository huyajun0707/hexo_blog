---
title: kotlin协程
tags: 
  - kotlin
categories:
  - [kotlin] 
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/dojm2h.png?w=1920&q=100&fmt=webp
excerpt: 协程并不是 Kotlin 提出来的新概念，其他的一些编程语言，例如：Go、Python 等都可以在语言层面上实现协程，甚至是 Java，也可以通过使用扩展库来间接地支持协程。「协程 Coroutines」源自 Simula 和 Modula-2 语言，这个术语早在 1958 年就被 Melvin Edward Conway 发明并用于构建汇编程序，说明协程是一种编程思想，并不局限于特定的语言。
---
# 协程是什么
协程并不是 Kotlin 提出来的新概念，其他的一些编程语言，例如：Go、Python 等都可以在语言层面上实现协程，甚至是 Java，也可以通过使用扩展库来间接地支持协程。

**最常见的说法：**

- Kotlin 官方文档说「本质上，协程是轻量级的线程」。
- 很多博客提到「不需要从用户态切换到内核态」、「是协作式的」等等。

「协程 Coroutines」源自 Simula 和 Modula-2 语言，这个术语早在 1958 年就被 Melvin Edward Conway 发明并用于构建汇编程序，说明协程是一种编程思想，并不局限于特定的语言。
当我们讨论协程和线程的关系时，很容易陷入中文的误区，两者都有一个「程」字，就觉得有关系，其实就英文而言，Coroutines 和 Thread就是两个概念。

**从 Android 开发者的角度去理解它们的关系：**

- 我们所有的代码都是跑在线程中的，而线程是跑在进程中的。
- 协程没有直接和操作系统关联，但它不是空中楼阁，它也是跑在线程中的，可以是单线程，也可以是多线程。
- 单线程中的协程总的执行时间并不会比不用协程少。
- Android 系统上，如果在主线程进行网络请求，会抛出 NetworkOnMainThreadException，对于在主线程上的协程也不例外，这种场景使用协程还是要切线程的。

**总结：** 协程是跑在线程上的，一个线程可以同时跑多个协程，每一个协程则代表一个耗时任务，我们手动控制多个协程之间的运行、切换，决定谁什么时候挂起，什么时候运行，什么时候唤醒，而不是 Thread 那样交给系统内核来操作去竞争 CPU 时间片。
协程设计的初衷是为了解决并发问题，避免回调地狱，让 「协作式多任务」 实现起来更加方便。

# 基本使用
## 一、添加依赖
//依赖当前平台所对应的平台库

```bash
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.3"
//依赖协程核心库
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.3.3"
```


Kotlin 协程是以官方扩展库的形式进行支持的。而且，我们所使用的「核心库」和 「平台库」的版本应该保持一致。

核心库中包含的代码主要是协程的公共 API 部分。有了这一层公共代码，才使得协程在各个平台上的接口得到统一。

平台库中包含的代码主要是协程框架在具体平台的具体实现方式。因为多线程在各个平台的实现方式是有所差异的。

## 二、简单使用

消除嵌套，在执行会线程操作后，会切回来

官方Demo中例子


```bash
fun main() {
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main thread continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
//输出结果
Hello,
World!
```
lauch开启了一个新协程，是可挂起的，在协程挂起时，会释放底层的线程，当协程协行完就会恢复。如果注释掉最后一行的， 则只打印Hello, 没有World。

上面例子中 delay方法是suspend函数，在thread中调用会报错，它只能在协程中使用。

上面代码输出线程


```bash
fun main(args: Array<String>) {
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("[${Thread.currentThread().name}]World!") // print after delay
    }
    println("[${Thread.currentThread().name}]Hello,") // main thread continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
//输出结果
[main]Hello,
[DefaultDispatcher-worker-1]World!
```
### suspend 挂起 （挂起的是协程）
协程在执行到有 suspend 标记的函数的时候，会被 suspend 也就是被挂起，而所谓的被挂起，就是切个线程；

不过区别在于，挂起函数在执行完成之后，协程会重新切回它原先的线程。

再简单来讲，在 Kotlin 中所谓的挂起，就是一个稍后会被自动切回来的线程调度操作。

#### 非阻塞式挂起

挂起指是在执行它（协程）的当前协程挂起，指当前协程停下来了，从当前代码起将不再运行这个协程了。

要求|限制：挂起函数，要么在协程里面被调用，要么在另一个挂起函数中被调用

作用：它其实是一个提醒，函数的创建者对函数的调用者的提醒

#### 什么时候使用suspend？
原则：耗时 （IO操作，比如文件的读写、网络操作、图片的模糊或者美化处理，特殊处理：等待几秒之后操作等）

[![tNpE6K.md.png](https://s1.ax1x.com/2020/06/02/tNpE6K.md.png)](https://imgchr.com/i/tNpE6K)

如果你创建一个 suspend 函数但它内部不包含真正的挂起逻辑，编译器会给你一个提醒：redundant suspend modifier，告诉你这个 suspend 是多余的。

因为你这个函数实质上并没有发生挂起，那你这个 suspend 关键字只有一个效果：就是限制这个函数只能在协程里被调用，如果在非协程的代码中调用，就会编译不通过。

所以，创建一个 suspend 函数，为了让它包含真正挂起的逻辑，要在它内部直接或间接调用 Kotlin 自带的 suspend 函数，你的这个 suspend 才是有意义的。

## 三、协程作用域
- GlobeScope 启动的协程单独启动一个协程作用域，内部的子协程遵从默认的作用域规则。通过 GlobeScope 启动的协程“自成一派”。
- coroutineScope 是继承外部 Job 的上下文创建作用域，该构造器会创建一个协程作用域，并且会等待所有启动的子协程全部完成后自身才会完成(注意：runBlocking与coroutineScope之间的主要区别在于coroutineScope在等待所有子协程完成其任务时并不会阻塞当前线程，而runBlocking会阻塞当前线程)。在其内部的取消操作是双向传播的，子协程未捕获的异常也会向上传递给父协程。它更适合一系列对等的协程并发的完成一项工作，任何一个子协程异常退出，那么整体都将退出，简单来说就是”一损俱损“。这也是协程内部再启动子协程的默认作用域。
- supervisorScope 同样继承外部作用域的上下文，但其内部的取消操作是单向传播的，父协程向子协程传播，反过来则不然，这意味着子协程出了异常并不会影响父协程以及其他兄弟协程。它更适合一些独立不相干的任务，任何一个任务出问题，并不会影响其他任务的工作，简单来说就是”自作自受“，例如 UI，我点击一个按钮出了异常，其实并不会影响手机状态栏的刷新。需要注意的是，supervisorScope 内部启动的子协程内部再启动子协程，如无明确指出，则遵守默认作用域规则，也即 supervisorScope 只作用域其直接子协程。

## 四、启动协程
- launch
- async
- runBlocking

### 1.  launch：返回Job
含义：我要创建一个新的协程，并在指定的线程上运行它。这个被创建、被运行的所谓「协程」就是你传给 launch 的那些代码，这一段连续代码叫做一个「协程」。

job.join()/job.cancel() -->可以用Job的join()方法来显式地等待这个协程结束；job.cancel()用于取消不再需要的协程任务.

GlobalScope.launch  使用 GlobalScope 单例对象,直接调用 launch 开启协程。


```bash
GlobalScope.launch {
    getImage(imageId)
}
```

与runBlocking区别在于不会阻塞线程。但在 Android 开发中同样不推荐这种用法，因为它的生命周期会和 app 一致，且不能取消。

coroutineScope.launch 自行通过 CoroutineContext 创建一个 CoroutineScope 对象，需要一个类型为 CoroutineContext 的参数。


```bash
val coroutineScope = CoroutineScope(coroutineContext)
coroutineScope.launch {
    getImage(imageId)
}
```

比较推荐的使用方法，我们可以通过 coroutineContext 参数去管理和控制协程的生命周期（这里的 context 和 Android 里的不是一个东西，是一个更通用的概念，会有一个 Android 平台的封装来配合使用）。

如果只是使用 launch 函数，协程并不能比线程做更多的事。不过协程中却有一个很实用的函数： **withContext**  。这个函数可以切换到指定的线程，并在闭包内的逻辑执行结束之后，自动把线程切回去继续执行。那么可以将上面的代码写成这样：

```bash
//通过launch
GlobalScope.launch {
    log(1)
    launch(Dispatchers.Main) {
        log(2)
        launch(Dispatchers.IO) {
            log(3)
            launch(Dispatchers.Main) {
                log(4)
            }
        }
    }
}
//withContext 由于可以"自动切回来"，消除了并发代码在协作时的嵌套。由于消除了嵌套关系
GlobalScope.launch(Dispatchers.Main) {
    withContext(Dispatchers.IO) {
        log(1)
    }
    log(2)
    withContext(Dispatchers.IO) {
        log(3)
    }
    log(4)
}
```

### 2. async: 从协程返回值
async 与 launch 从功能上是同等类型的函数，它们都被称作协程的 Builder 函数，不同之处在于async开启线程, 返回Deferred<T>, Deferred<T>是Job的子类, 有一个await()函数, 可以返回协程的结果.


```bash
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val one = async { searchItemOne() }
        val two = async { searchItemTwo() }
        println("The items is ${one.await()} and ${two.await()}")
    }
    println("cost time is ${time} ms")
}

suspend fun searchItemOne(): String {
    delay(1000L)
    return "item-one"
}

suspend fun searchItemTwo(): String {
    delay(1000L)
    return "item-two"
}
//输出结果
The items is item-one and item-two
cost time is 1019 ms
```

### 3. runBlocking 

主协程，launch创建的协程能够在runBlocking中运行，反之不行.

可以建立一个阻塞当前线程的协程. 所以当前线程会一直 阻塞 直到 runBlocking 内部的协程执行完毕（也就是说，调用它的线程会一直位于该函数中，直到协程执行完毕为止），适用于单元测试的场景，而业务开发中不会用到这种方法，因为它是线程阻塞的。

 
```bash
fun main(args: Array<String>)= runBlocking {
    launch {
       delay(1000L)
        println("World!")
    }
    println("Hello, ")

}
//执行结果
Hello, 
World!
```

## 四、协程相关概念
### 启动参数

- CoroutineContext       协程的上下文，用于在协程与协程之间参数传递，EmptyCoroutineContext 表示一个空的协程上下文
- CoroutineStart         协程的启动方式，四种标准方式
- block: suspend CoroutineScope.() -> Unit   协程真正要去执行的内容
- parent: Job        表示在当前 协程 闭包的外层的 job，一般情况下使用不到

#### CoroutineContext 
coroutineContext 协程上下文包含当前协程scope的信息， 比如的Job, ContinuationInterceptor, CoroutineName 和CoroutineId。在CoroutineContext中，是用map来存这些信息的， map的键是这些类的伴生对象，值是这些类的一个实例。


```bash
//可以为协程指定上下文添加名称，如果有多个上下文需要添加，可直接用+
GlobalScope.launch(CoroutineName("Test")) {
  println("-->${coroutineContext[CoroutineName]}")
}
//输出结果
-->CoroutineName(Test)

GlobalScope.launch(Dispatchers.Main + CoroutineName("Hello")) {
   println("-->${ Thread.currentThread().name}")
}
//输出结果
-->main
```

#### CoroutineStart    哪种方式启动

模式| 功能
---|---
DEFAULT | 立即执行，表示当前线程什么时候有空，就什么时候启动
ATOMIC | 立即执行协程体，但在开始运行之前无法取消
UNDISPATCHED |  立即在当前线程执行协程体，直到第一个 suspend 调用 挂起之后的执行线程取决于上下文当中的调度器了
LAZY | 只有在需要的情况下运行,如果没有手动调用 job 对象的 start() 或 join() 方法的话，那么该 协程 是不会被启动的


```bash
suspend fun main() {
    log(1)
    val job = GlobalScope.launch(start = CoroutineStart.LAZY) {
        log(2)
    }
    log(3)
    job.start()
    log(4)
    Thread.sleep(1000L)
}
//输出结果
--->1
--->3
--->4
--->2
```

#### ContinuationInterceptor拦截器

所有协程启动的时候，都会有一次 Continuation.resumeWith 的操作

```bash
class MyContinuationInterceptor: ContinuationInterceptor {
    override val key = ContinuationInterceptor
    override fun <T> interceptContinuation(continuation: Continuation<T>) = MyContinuation(continuation)
}

class MyContinuation<T>(val continuation: Continuation<T>): Continuation<T> {
    override val context = continuation.context

    override fun resumeWith(result: Result<T>) {
        println("<MyContinuation> $result" )
        continuation.resumeWith(result)
    }
}
```


```bash
suspend fun main() {
    GlobalScope.launch(MyContinuationInterceptor()) {
        log(1)
        val job = async {
            log(2)
            delay(1000)
            log(3)
            "Hello"
        }
        log(4)
        val result = job.await()
        log("5. $result")
    }.join()
    log(6)
}
//输出结果
15:31:55:989 [main] <MyContinuation> Success(kotlin.Unit)  // ①
15:31:55:992 [main] 1
15:31:56:000 [main] <MyContinuation> Success(kotlin.Unit) // ②
15:31:56:000 [main] 2
15:31:56:031 [main] 4
15:31:57:029 [kotlinx.coroutines.DefaultExecutor] <MyContinuation> Success(kotlin.Unit) // ③
15:31:57:029 [kotlinx.coroutines.DefaultExecutor] 3
15:31:57:031 [kotlinx.coroutines.DefaultExecutor] <MyContinuation> Success(Hello) // ④
15:31:57:031 [kotlinx.coroutines.DefaultExecutor] 5. Hello
15:31:57:031 [kotlinx.coroutines.DefaultExecutor] 6
```

首先，所有协程启动的时候，都会有一次 Continuation.resumeWith 的操作，这一次操作对于调度器来说就是一次调度的机会，我们的协程有机会调度到其他线程的关键之处就在于此。 ①、② 两处都是这种情况。

其次，delay 是挂起点，1000ms 之后需要继续调度执行该协程，因此就有了 ③ 处的日志。
最后，④ 处的日志就很容易理解了，正是我们的返回结果。

## 四、协程操作
lauch启动协程会返回一个Job对象，用来操作协程

###  join方法
为了能够让程序在协程执行完毕之前一直保活！
以非阻塞方式等待后台job执行

```bash
val job = GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
    delay(1000L)
    println("World!")
}
job.join() // 等待直到子协程执行结束
println("Hello,")
//输出结果，先执行job协程中的代码
World!
Hello,
```
### cancel 取消
cancelAndJoin和cancel并不会真正取消，需要在协程中判断 isActive，isActive 是一个可以被使用在 CoroutineScope 中的扩展属性。

```bash
val startTime = System.currentTimeMillis()
val job = launch(Dispatchers.Default) {
    var nextPrintTime = startTime
    var i = 0
    while (isActive) { // 可以被取消的计算循环
        // 每秒打印消息两次
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
delay(1300L) // 等待一段时间
println("main: I'm tired of waiting!")
job.cancelAndJoin() // 取消该作业并等待它结束
println("main: Now I can quit.")
```

对于取消，除了 supervisorScope 比较特别是单向取消，即父协程取消后子协程都取消，Android 中 MainScope 就是一个调度到 UI 线程的 supervisorScope；coroutineScope 的逻辑则是父子相互取消的逻辑；而 GlobalScope 会启动一个全新的作用域，与它外部隔离，内部遵循默认的协程作用域规则。

## 五、调度器
Coroutine dispatchers 可以指定协程运行在 Android 的哪个线程里。主要有以下几类

- Default: CoroutineDispatcher
- Main: MainCoroutineDispatcher
- Unconfined: CoroutineDispatcher

-- | Jvm|Js|Native
---|---|---|---
Default |线程池 |主线程循环 |主线程循环
Main| UI 线程 |与 Default 相同 |与 Default 相同
Unconfined |直接执行 |直接执行 |直接执行
IO |线程池| –| –

- IO 仅在 Jvm 上有定义，它基于 Default 调度器背后的线程池，并实现了独立的队列和限制，因此协程调度器从 Default 切换到 IO 并不会触发线程切换。
- Main 主要用于 UI 相关程序，在 Jvm 上包括 Swing、JavaFx、Android，可将协程调度到各自的 UI 线程上。
- Js 本身就是单线程的事件循环，与 Jvm 上的 UI 程序比较类似。

```bash
suspend fun main() {
    val myDispatcher= Executors.newSingleThreadExecutor{ r -> Thread(r, "MyThread") }.asCoroutineDispatcher()
    GlobalScope.launch(myDispatcher) {
        log(1)
    }.join()
    log(2)
}
//关闭线程池
myDispatcher.close()
```
此处注意，在多个线程上运行协程，delay会切线程，协程挂起继续操作也会切线程，

## 六、异常捕获
CoroutineExceptionHandler  只能捕获对应协程内未捕获的异常

```bash
suspend fun main() {
    val exceptionHandler = CoroutineExceptionHandler { coroutineContext, throwable ->
        println("Throws an exception with message: ${throwable.message}")
    }

    log(1)
    GlobalScope.launch {
        throw ArithmeticException("Hey!")
    }.join()
    log(2)
}
```


1. 协程内部异常处理流程：launch 会在内部出现未捕获的异常时尝试触发对父协程的取消，能否取消要看作用域的定义，如果取消成功，那么异常传递给父协程，否则传递给启动时上下文中配置的 CoroutineExceptionHandler 中，如果没有配置，会查找全局（JVM上）的 CoroutineExceptionHandler 进行处理，如果仍然没有，那么就将异常交给当前线程的 UncaughtExceptionHandler 处理；而 async 则在未捕获的异常出现时同样会尝试取消父协程，但不管是否能够取消成功都不会后其他后续的异常处理，直到用户主动调用 await 时将异常抛出。
2. 异常在作用域内的传播：当协程出现异常时，会根据当前作用域触发异常传递，GlobalScope 会创建一个独立的作用域，所谓“自成一派”，而 在 coroutineScope 当中协程异常会触发父协程的取消，进而将整个协程作用域取消掉，如果对 coroutineScope 整体进行捕获，也可以捕获到该异常，所谓“一损俱损”；如果是 supervisorScope，那么子协程的异常不会向上传递，所谓“自作自受”。
3. join 和 await 的不同：join 只关心协程是否执行完，await 则关心运行的结果，因此 join 在协程出现异常时也不会抛出该异常，而 await 则会；考虑到作用域的问题，如果协程抛异常，可能会导致父协程的取消，因此调用 join 时尽管不会对协程本身的异常进行抛出，但如果 join 调用所在的协程被取消，那么它会抛出取消异常，这一点需要留意。

# 实现原理
- CPS 变换
- 续体与续体拦截器
- 状态机
## CPS 变换
CPS变换又叫做continuation Passing Style，它是一种编程风格,用来将内部要执行的逻辑封装到一个闭包里面,然后再返回给调用者,这就将它的程序流程显式的暴露给了程序员,让我们可以控制它。
kotlin协程中挂起函数或挂起 lambda 表达式调用时，都有一个隐式的参数额外传入，这个参数是Continuation类型，封装了协程恢复后的执行的代码逻辑。

声明一个suspend函数

```bash
suspend fun searchItemOne()
```

转成java代码如下

```bash
public static final Object searchItemOne(@NotNull Continuation $completion)
```

Continuation的定义如下，类似于一个通用的回调接口：

```bash
@SinceKotlin("1.3")
public interface Continuation<in T> {
    /**
     * Context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

Continuation中文为续体，简单来说它包装了协程在挂起之后应该继续执行的代码；在编译的过程中，一个完整的协程被分割切块成一个又一个续体。在函数的挂起结束以后，它会调用 continuation 参数的 resumeWith 函数，来恢复执行后面的代码

## 续体与续体拦截器
挂起函数在恢复的时候，理论上可能会在任何一个线程上恢复，有时我们需要限定协程运行在指定的线程，例如在 UI 编程中，更新 UI 的操作通常只能在 UI 主线程中进行，就需要进行线程切换，可通过拦截器中实现。

## 状态机
再看主函数

```bash
//主函数
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val one = async { searchItemOne() }
        val two = async { searchItemTwo() }
        println("The items is ${one.await()} and ${two.await()}")
    }
    println("cost time is ${time} ms")
}

suspend fun searchItemOne(): String {
    delay(1000L)
    return "item-one"
}

suspend fun searchItemTwo(): String {
    delay(1000L)
    return "item-two"
}
```

协程内部实现不是使用普通回调的形式，而是使用状态机来处理不同的挂起点

```bash
public static final void main(@NotNull String[] args) {
    ....
    public final Object invokeSuspend(@NotNull Object $result) {
        switch(this.label) {
            case 0:
                //简化后
                 this.label = 1;
                 searchItemOne();
                break;
            case 1:
                this.label = 2;
                 searchItemTwo();
                break;
            case 2:
               println("The items is ${one.await()} and ${two.await()}")
                break;
            default:
                throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
        }
    }
}
```

上面代码中每一个挂起点和初始挂起点对应的 Continuation 都会转化为一种状态，协程恢复只是跳转到下一种状态中。挂起函数将执行过程分为多个 Continuation 片段，并且利用状态机的方式保证各个片段是顺序执行的。

[![tNpAl6.png](https://s1.ax1x.com/2020/06/02/tNpAl6.png)](https://imgchr.com/i/tNpAl6)

协程就是这样，在一个线程中顺序执行的，先执行一段程序，这里用continuation表示，然后遇到suspension point是，程序悬挂，进行下一个continuation子程序的运行。

# 扩展
retrofit、room、anko、viewModel等一些库现已支持协程

## retrofit中使用
### 1. callAdapter


```bash
implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter:0.9.2'
```


#### retrofit配置


```bash
/**获取协程Retrofit配置信息*/
fun getRetrofitCoroutineConfig(): Retrofit.Builder {
    return Retrofit.Builder()
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(CoroutineCallAdapterFactory())
}
```

```bash
@GET("api/indexStatus")
fun getIndexStatus(): Deferred<BaseResponse<IndexStatus>>
```

```bash
GlobalScope.launch(Dispatchers.Main) {
    try {
        var result = ApiService.instance.getIndexStatus().await()

        println(Thread.currentThread().name + ":${result.data.toString()}")
    } catch (e: Exception) {
        Toast.makeText(BaseApplication.instance(), e.message, Toast.LENGTH_SHORT).show()
    }
}
```

### 2. suspend
retrofit不需要配置CoroutineCallAdapterFactory


```bash
@GET("api/indexStatus")
suspend fun getIndexStatus(): BaseResponse<IndexStatus>
```

```bash
GlobalScope.launch(Dispatchers.Main) {
    try {
        var result = ApiService.instance.getIndexStatus()

        println(Thread.currentThread().name + ":${result.data.toString()}")
    } catch (e: Exception) {
        Toast.makeText(BaseApplication.instance(), e.message, Toast.LENGTH_SHORT).show()
    }
}
```

## viewModel中使用

### 引入依赖


```bash
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0-rc03'
```


### 使用

```bash
viewModelScope.launch {
   value.value= ApiService.instance.getIndexStatus().data.toString()
}
```



#### 参考
https://www.jianshu.com/p/d23c688feae7
https://www.bennyhuo.com/
https://ethanhua.github.io/2018/12/24/kotlin_coroutines/


