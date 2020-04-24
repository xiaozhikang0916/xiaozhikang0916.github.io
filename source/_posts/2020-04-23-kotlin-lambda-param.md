---
title: kotlin 中传入形参、返回不匹配的 lambda 表达式
date: 2020-04-23 21:42:52
tags:
    - kotlin
    - lambda
---

公司的项目里已经大范围应用上了 kotlin ，其 `lambda` 函数以及高阶函数的特性让我们在开发中享受了不少便利。

最近在 lambda 的使用上遇到了一点小问题，借此稍微探究了一下 kotlin 中 lambda 的小细节，并做一个记录。

## 起因

在我的代码中，经常需要遍历处理一个 list ，并且将经过处理的数据添加到另一个 list 中，示例如下：

```kotlin
list.map { wrap(it) }.forEach { result.add(it) }
```

其中，数据流的最后一个调用 `forEach` 接受一个以数据类型为入参的函数，正好与 `ArrayList.add(T)` 匹配，那么我便想改写成以下形式，简化代码：

```kotlin
list.map(::wrap).forEach(result::add)
```

结果得到编译错误：

>Error:(11, 22) Kotlin: Type inference failed: inline fun \<T> Iterable\<T>.forEach(action: (T) -> Unit): Unit
cannot be applied to
receiver: List\<String>  arguments: (\<unknown>)
>
>Error:(11, 38) Kotlin: None of the following functions can be called with the arguments supplied:
public open fun add(index: Int, element: String): Unit defined in java.util.ArrayList
public open fun add(element: String): Boolean defined in java.util.ArrayList

经过一番思考，我认为是一个类型为 `((T) -> Boolean)` 的函数不能作为实参，传给形参类型为 `((T) -> Unit)` 的函数。

在忍住了 “ 你想要 Unit ， 我返回了 Boolean ，你不去用不就好了 ” 的吐槽后，我开始思考函数作为参数传入的话，有什么规则与限制。

## 首次尝试

首先我想先确认，作为实参的函数的形参与返回类型是否必须与高阶函数中作为参数的函数的形参与返回类型完全一致，来一段简单的代码：

```kotlin
fun high(foo: ((Int) -> CharSequence)): CharSequence {
    return foo.invoke(1)
}

fun main(args: Array<String>) {
    val wrap : ((Number) -> String) = { (10 + it.toLong()).toString() }
    println(high(::wrap))
}
```

点击运行，程序正常编译通过，输出为 11 也符合预期，那么似乎可以得到初步的结论：

1. 实参函数的形参与返回类型不必与形参函数的形参与返回类型完全一致；
2. 实参函数的形参类型可以是形参函数的形参类型的超类；
3. 实参函数的返回类型可以是形参函数的返回类型的子类。

上面几点结论，仔细观察的话，会发现与 Java 和 kotlin 中泛型的协变、逆变有相似之处，那么其中有何关联？我便想到了去编译后字节码一探究竟。

## 窥探编码

使用 IDEA 集成的 `Show Kotlin Bytecode` 工具，我们可以很方便地实时查看到当前 kotlin 文件所对应的编译后字节码。

直接来到我们的高阶函数 `high` 对应的函数入口，发现其入口是：

`public final static high(Lkotlin/jvm/functions/Function1;)Ljava/lang/CharSequence;`

明显可以看到，函数的入参类型是 `Function1` 。我们知道在 kotlin 的定义中，`Function1` 对应的是形参列表有且只有一个的 lambda 表达式。在字节码展示窗口的底部，我们能看到上面所写的 lambda 表达式 `wrap` 所对应生成的内部类：

`final class TestLambdaKt$main$wrap$1 extends kotlin/jvm/internal/Lambda  implements kotlin/jvm/functions/Function1`

很显然了，这个 lambda 表达式所对应生产的内部类是一个 `Lambda` 的子类，并且实现了 `Function1` 接口。

结合 `high` 函数的入口，我们知道了 kotlin 中的高阶函数并没有限定传入函数的形参与返回类型完全一致，而是用其对应的 `FunctionX` 接口作为形参，以保证使用时足够的自由度。

那么简单的脑洞，是不是任何一个 `Function1` 的 lambda 表达式都能传入？

我们把 wrap 修改一下：

```kotlin
val wrap : ((Number) -> Unit) = { (10 + it.toLong()).toString() }
```

...显然会报错，不然最开始的 `ArrayList.add(T)` 就不会有问题了。

再仔细看字节码，发现编译器很贴心地为我们留下了注释，其中对应 `high` 函数入口的注释是：

> // declaration: java.lang.CharSequence high(kotlin.jvm.functions.Function1<? super java.lang.Integer, ? extends java.lang.CharSequence>)

对应 lambda 表达式生成的内部类的注释是：

> // declaration: com/bilibili/test/TestLambdaKt$main$wrap$1 extends kotlin.jvm.internal.Lambda implements kotlin.jvm.functions.Function1<java.lang.Number, java.lang.String>

果然是与泛型有关，其中函数入口的形参类型是逆变的，返回类型是协变的，也证明了上面我的猜测是正确的。

而 `ArrayList.add(T)` 不能作为方法引用传给 `forEach` 的原因，就是 `add` 返回了 `Boolean` 的返回值，而 `forEach` 要求传入函数的返回值是 `Unit`，不会有任何的子类，因此不满足泛型的限制。

### 传递一个方法引用

在 kotlin 中，方法引用可以说跟 lambda 表达式有着同样的地位，都是高阶函数的一环。那么上面的示例代码同样可以修改为如下方法引用的形式：

```kotlin
fun main(args: Array<String>) {
    println(high(::wrap))
}

fun wrap(i: Number): String {
    return (10 + i.toLong()).toString()
}

fun high(foo: ((Int) -> CharSequence)): CharSequence {
    return foo.invoke(1)
}
```

这时，高阶函数 `high` 在字节码中的入口没有改变，而方法引用语句 `::wrap` 生成了一个继承自 `kotlin/jvm/internal/FunctionReference` 并且实现了 `Function1` 的内部类。因为同样实现了 `Function1` 接口，便保证了与之前 lambda 表达式示例代码有相同的表现。

其字节码的具体内容，有兴趣的朋友可以自行查看。

## 总结

通过查看字节码，我了解到了 kotlin 内部是使用了泛型接口，来保证方法引用、lambda 表达式与高阶函数的泛用性的，同时捎带复习了一下 Java 中协变、逆变的知识，以后在使用高阶函数时更心里有数了。

### bonus

在最开始尝试的代码里，我有过如下尝试：

```kotlin
val fun1 : ((Int) -> CharSequence) = {""}
val fun2 : ((Int) -> String) = {""}

println(fun1.javaClass.isAssignableFrom(fun2.javaClass))
```

显然结果是 `false` ，他们不是父类子类的关系。

并且每写一句 lambda 表达式，即使他们的类型和语句都完全一样，编译器都会为他们单独生成一个内部类，对编译结果体积有要求的话需要注意了。
