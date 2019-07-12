---
title: "Kotlin 中 forEach 的中断"
date: 2019-07-11 18:37:08
tags: 
    - kotlin
    - forEach
    - java 
    - stream
---

## 想要一个 `break`

相较于原生的 `for` 关键字， kotlin （包括 1.8 版本后的 Java） 为集合引入了符合函数式编程范式的 `forEach` 方法，提升了代码编写的体验。

```kotlin
val data = List(5) { it.toString() }
data.forEach {
    println("Parsing $it")
    val parsedData = it.toInt()
}
```

但是，随着业务发展，我发现需要在以前的 `forEach` 循环中加入条件，满足某些条件后直接退出循环，也就是说需要一个 `break` 关键字。

## `forEach` 有没有 `break`

查看 kotlin 库中 `forEach` 的源码会发现， `forEach` 并不是一个原生的关键字，而是库为我们包装好的内联函数：

```kotlin
public inline fun <T> Iterable<T>.forEach(action: (T) -> Unit): Unit {
    for (element in this) action(element)
}
```

而我们在花括号中传入的代码块最后会被处理为 lambda 函数传入，在其内部的 `for` 语句中实现循环；而在一段函数中当然不能中断函数外的循环了。

## 怎么实现一个 `break`

既然库中没有提供一个 `break` 关键字，那么我们当然要自己造一个轮子出来了。

### 1. 返回这个方法

查看源码后的第一反应，满足中断条件就应当是不执行后面的处理逻辑，那么直接使用返回不就好了吗？

```kotlin
val data = List(5) { it.toString() }
var parsed = 0
data.forEach {
    if (it.toInt() > 2) {
        return@forEach
    }
    parsed += 1
}
assertEquals(3, parsed) // 0 1 2
```

然而仔细考虑就会发现这种其实很有问题：循环并不是真正地被中断，完整地走完了循环，只是部分数据没有进行操作而已：

```kotlin
val data = List(5) { it.toString() }
var before = 0
var after = 0
data.forEach {
    before += 1
    if (it.toInt() > 3) {
        return@forEach
    }
    after += 1
}
assertEquals(3, after) // 0 1 2
assertEquals(3, before) // FAILED!!! actually 5
```

这是 `Continue` ， 不是 `break`。

### 2. 暴力中断

`break` 中断本质是想要从循环体中直接退出到循环外，那么最简单的办法...

```kotlin
class LoopBreakException: Throwable()

val data = List(5) { it.toString() }
var before = 0
var after = 0
try {
    data.forEach {
        before += 1
        if (it.toInt() > 2) {
            throw LoopBreakException()
        }
        after += 1
    }
} catch (e: LoopBreakException) {
}
assertEquals(4, before)
assertEquals(3, after)
```

又不是不能用.jpg ，系统中断不也是这个思路吗？

### 3. 不提供不需要处理的数据

既然 *不处理不想要的数据* 行不通，那么我们应当换一个思路，考虑 *不提供不需要处理的数据* 这种做法。

首先复习一下 kotlin 给我们提供的集合的操作方法，知道它是可以通过链式调用 `map` `filter` `takeWhile` 等方式对数据进行预处理的，而 `takeWhile` 就是我们需要的：

```kotlin
    val data = List(5) { it.toString() }
    var before = 0
    var after = 0
    data.takeWhile {
        before += 1
        it.toInt() <= 2
    }.forEach {
        it.toInt() // let's say you are parsing data with toInt()
        after += 1
    }
    assertEquals(3, after) // 0 1 2
    assertEquals(4, before) // 多一次判断，不符合条件后停止
```

可以看到循环次数确实有了减少。

但是在示例中，为了判断是否中断需要解析数据（`String.toInt()`），在实际处理中又解析了一次。为了减少重复代码和处理消耗，我们应该充分利用链式调用，提前处理好数据：

```kotlin
    val data = List(5) { it.toString() }
    var parsing = 0
    var before = 0
    var after = 0
    data.map {
        parsing += 1
        it.toInt()
    }.takeWhile {
        before += 1
        it <= 2
    }.forEach {
        doYourBiz(it) // let's say you are parsing data with toInt()
        after += 1
    }
    assertEquals(3, after) // 0 1 2
    assertEquals(4, before)// 多一次判断，不符合条件后停止
    assertEquals(5, parsing)
```

在这里会发现，因为数据处理发生在 `takeWhile` 之前，而 `Collection` 的操作是先循环执行完前一段，再用结果作为新的集合去循环执行下一段，显然会有时间与内存上的消耗。

这里可以用 `Sequence` 做进一步的优化：

```kotlin
val data = List(5) { it.toString() }
var parsing = 0
var before = 0
var after = 0
data.asSequence()
    .map {
    parsing += 1
    it.toInt()
}.takeWhile {
    before += 1
    it <= 2
}.forEach {
    doYourBiz(it) // let's say you are parsing data with toInt()
    after += 1
}
assertEquals(3, after) // 0 1 2
assertEquals(4, before)// 多一次判断，不符合条件后停止
assertEquals(4, parsing)// 同样是4次，停止后也不再执行后面的处理
```

> 对 `Collection` 和 `Sequence` 的链式处理逻辑感兴趣的同学，可以自行将 parsing before after 打印出来，观察其中的区别。

## 结语

归根结底，一开始想要在 `forEach` 中使用 `break` 是自己的观念没有转变好：在这种链式调用处理数据的函数式编程场景，仍然抱着指令式编程的观念。在这种情况下应该细分所需的业务逻辑，将其归类分层，才能写出简洁直观的代码。

```kotlin
// forEach前
for(data in dataList) {
    val parsed = parseData(data)
    if (shouldBreak(parsed)) {
        break
    } else {
        yourBiz(parsed)
    }
}

// forEach后
dataList.asSequence()
    .map {
        parseData(it)
    }.takeWhile {
        !shouldBreak(it)
    }.forEach {
        yourBiz(it)
    }
```

Java 1.8 后提供的 `stream` 流也有上述类似的概念，本文也可供参考。
