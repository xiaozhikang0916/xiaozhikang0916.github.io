---
title: 使用协程组合管理业务逻辑
date: 2021-12-25 17:17:54
tags:
    - Android
    - coroutine
    - 安卓
    - 协程
---

在我们的项目中，已经开始推广协程的使用了，其本身的特性让我们在编写并发、异步、后台逻辑时，获得了比其他如 `bolt.Tasks` 等第三方库更好的开发体验。
在此我将总结一些在业务场景中使用协程的经验。

## 一个协程框架下的功能怎么写

在使用协程编写业务逻辑之前，我们在耗时任务上使用的三方库、组件API、甚至于说编程思想都是基于回调的概念的：我们从在调用时发起一个异步任务，为其注册好监听事件，随机调用函数返回但没有结果。

在异步任务有任何需要自己关心的事件发生时，回调我们注册的监听，如在 `onUpdate` 里更新进度，在 `onSuccess` 里获取结果，有可能需要再开启一个新的异步任务，在 `onFail` 里处理失败状态、取消仍未完成的其他任务。

在这种框架之下有诸多不便，如不同子任务的时序关系难以确定；不同异步任务的状态难以管理、容易内存泄漏；回调api的设计有理解成本（如，回调执行在什么线程）等。在[相关阅读](#相关阅读)中有其他博客对此有进一步的描述。

而在刚接触协程编程时，之前的经验有可能会成为编写简洁协程代码的限制。

### 将思想同步回同步

在使用协程编写业务逻辑时，一个要做的重要思想转变是：在最外层的业务逻辑里，调用应该**在形式上**是同步调用的，任何调用语句的返回都意味着一个任务的执行完成，可以明确获得其结果进行后续的逻辑，其中的具体实现是使用挂起还是阻塞，都由其中子任务的函数封装保证。

### 不要害怕抛出异常

平时在做业务开发的工作中，我们似乎养成一种观念：抛出异常是很可怕的事情，意味着崩溃、意味着线上事故。

然而在协程的框架中，函数的返回应该只用于回传结果数据；协程的结构性并发、上下文的取消设计也依赖于内部抛出的异常；在复杂的业务代码下，`suspend` 函数的调用会嵌套多层，抛出异常是能将错误信息通知到调用栈中所有成员的最简单方式。

因此，在编写协程的子任务逻辑时，我们要做的并不是找出一种 `onFail` 的替代方式，而是把自己的错误信息封装成一个合适的异常，将其抛出；然后在合适的代码位置捕获，进行处理。

最后，为自己养成良好的使用 `try-catch` 的习惯。

> 实在不行还能用 `CancellationException` 包一层。

> 实在不行还能用 `CoroutineExceptionHandler` 兜住。

### 业务代码关注主流程：One thing to achieve

在所有的子任务都有了合适的挂起函数的封装后，最外层的业务逻辑会变成一件水到渠成的事情：我们只需要按顺序调用所有的子任务拿到结果就可以了：

```kotlin

suspend fun subTask1(): Int

suspend fun subTask2(): String

suspend fun uploadString(str: String)

suspend fun mainTask() {
    try {
        if (subTask1() > 5) {
            val str = subTask2()
            uploadString(str)
        }
        // Task done successfully
    } catch (e: UploadExcption) {
        // some cleaning job
    }
}

```

在 `mainTask` 的主要代码块中，我们只需要按照**所有任务正常执行**的逻辑，编写出业务代码即可；意即，如果某一个语句的调用依赖于前一个任务的正常执行的结果，那么在调用这个语句的时候不需要判断前面任务是否成功，而是**默认**其已经正常执行返回、获得了有意义的结果。

因此主要代码块中的所有语句都是为了完成业务逻辑而编写的 `One thing to achieve` ，不去关心进度处理、错误逻辑等边界情况。

而在有任务失败时，会默认因为它的失败需要取消掉该业务逻辑，因此跳出了该 `try` 块、后续语句都不再执行，并按错误类型进行特殊或者统一的处理和状态清理。

## 一个假想的业务场景

以一个假想的业务场景为例：

假设在我们的一个业务功能中，可以拆分为4个独立存在的耗时子任务；其中前三个子任务互相独立，可以并发执行；第四个任务需要等待前三个均成功完成之后再开始执行；四个子任务都完成之后业务功能正常返回、显示成功，否则显示具体失败的任务，并做好清理工作。

### 容易失败的子任务

在这个业务中，首先为各个子任务定义状态、错误类型，然后编写核心逻辑：

```kotlin
enum class TaskStatus {
    Init,
    Loading,
    Succeed,
    Failed
}

class TaskException(id: Long) : IOException("$id is failed")

suspend fun startSubTask(id: Long): Long {
    Log.i(TAG, "Start task of $id")
    try {
        delay((id + 1) * 500)
    } catch (e: CancellationException) {
        // 并发执行的其他任务失败了，此任务不再继续，直接取消
        Log.i(TAG, "Task $id cancelled by others", e)
        throw e
    }
    val rand = Random.nextInt(3)
    Log.i(TAG, "Task $id random $rand")
    if (rand > 0) {
        Log.i(TAG, "Task $id success, returning")
        return id * rand
    }
    Log.i(TAG, "Task $id failed, throwing")
    throw TaskException(id)
 }
```

### 统筹整体逻辑的主任务

主任务的逻辑中，主要用来管理子任务的逻辑与依赖关系，并向 view 层传递结果状态：

```kotlin
data class MainTask(
    val status: TaskStatus,
    val cause: Throwable? = null
)

val mainTask = MutableStateFlow(MainTask(TaskStatus.Init))
suspend fun startMainTask() {
    mainTask.emit(MainTask(TaskStatus.Loading))
    try {
        coroutineScope {
            awaitAll(
                async { startSubTask(1) },
                async { startSubTask(2) },
                async { startSubTask(3) },
            )
        }
        val lastTask = startSubTask(4)
        Log.i(TAG, "Last task result $lastTask")
        mainTask.emit(MainTask(TaskStatus.Succeed))
    } catch (e: Exception) {
        // 有子任务失败，此处不太关心具体是谁失败了，只需要通知失败状态
        mainTask.emit(MainTask(TaskStatus.Failed, e))
    }
}
```

### 跑一跑

在上述的代码下，可以方便地看到各任务的执行情况。

#### All succeed

UI: `Succeed`

log:

```log
I/TaskLog: Start task of 1
I/TaskLog: Start task of 2
I/TaskLog: Start task of 3
I/TaskLog: Task 1 random 1
I/TaskLog: Task 1 success, returning
I/TaskLog: Task 2 random 2
I/TaskLog: Task 2 success, returning
I/TaskLog: Task 3 random 2
I/TaskLog: Task 3 success, returning
I/TaskLog: Start task of 4
I/TaskLog: Task 4 random 1
I/TaskLog: Task 4 success, returning
I/TaskLog: Last task result 4
```

#### 1~3 失败

UI: `1 is failed`

log:

```log
I/TaskLog: Start task of 1
I/TaskLog: Start task of 2
I/TaskLog: Start task of 3
I/TaskLog: Task 1 random 0
I/TaskLog: Task 1 failed, throwing
I/TaskLog: Task 2 cancelled by others
    kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=ScopeCoroutine{Cancelling}@93967da
    Caused by: site.xiaozk.demo.coroutine_task.TaskException: 1 is failed
        at site.xiaozk.demo.coroutine_task.MainViewModel.startSubTask(MainViewModel.kt:55)
        ...
I/TaskLog: Task 3 cancelled by others
    kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=ScopeCoroutine{Cancelling}@93967da
    Caused by: site.xiaozk.demo.coroutine_task.TaskException: 1 is failed
        at site.xiaozk.demo.coroutine_task.MainViewModel.startSubTask(MainViewModel.kt:55)
        ...
```

需要注意的是，如果并发的任务有成功的，后续的失败不会再将其取消：

```log
I/TaskLog: Start task of 1
I/TaskLog: Start task of 2
I/TaskLog: Start task of 3
I/TaskLog: Task 1 random 2
I/TaskLog: Task 1 success, returning
I/TaskLog: Task 2 random 0
I/TaskLog: Task 2 failed, throwing
I/TaskLog: Task 3 cancelled by others
    kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=ScopeCoroutine{Cancelling}@190857e
    Caused by: site.xiaozk.demo.coroutine_task.TaskException: 2 is failed
        at site.xiaozk.demo.coroutine_task.MainViewModel.startSubTask(MainViewModel.kt:55)
        ...
```

#### 任务4失败

UI: `4 is failed`

log:

```log
I/TaskLog: Start task of 1
I/TaskLog: Start task of 2
I/TaskLog: Start task of 3
I/TaskLog: Task 1 random 2
I/TaskLog: Task 1 success, returning
I/TaskLog: Task 2 random 1
I/TaskLog: Task 2 success, returning
I/TaskLog: Task 3 random 1
I/TaskLog: Task 3 success, returning
I/TaskLog: Start task of 4
I/TaskLog: Task 4 random 0
I/TaskLog: Task 4 failed, throwing
```

## 相关阅读

[示例demo](https://github.com/xiaozhikang0916/coroutine-task-demo)
文中的实例代码已上传为demo项目，点击按钮即可随机看到成功或者失败的输出信息。

[《谈谈 Kotlin 协程的 Context 和 Scope》](https://blog.yujinyan.me/posts/kotlin-coroutine-context-scope/)以及其中的引用文章描述了协程的一些基本概念。

[《结构性并发与回调》](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)描述了普通的回调、`goto`语句的有害性。
