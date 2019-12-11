---
title: Android 使用Powermock的单元测试的 Rule 顺序
date: 2019-12-10 21:00:28
tags:
    - Android
    - Unittest
    - Powermock
    - mock rule
    - RuleChain
---

在使用 [Powermock](https://github.com/powermock/powermock) 进行安卓的单元测试的编写时，出现了一些已经被调用了初始化的业务代码在执行测试时没有被初始化的情况，在此记录一下解决问题的过程。

## 初始项目

首先建立一个类 `InitData` ，代表需要被初始化的业务单例代码：

```kotlin
object InitData {
    private var initData: Int = 0

    fun init(init: Int) {
        initData = init
    }

    fun getData(): Int {
        return initData
    }
}
```

然后建立一个测试，需要在测试开始前对这个单例进行初始化，然后验证传入的值：

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [26])
class BeforeTest {
    @Rule
    @JvmField
    val powerMockRule = PowerMockRule()

    @Before
    fun init() {
        InitData.init(10)
    }

    @Test
    fun testInit() {
        assertEquals(10, InitData.getData())
    }
}
```

## 测试优化

像上面的代码能正常运行通过，但是：

1. 大部分的初始化操作都很类似，容易出现代码复制粘贴的情况；
2. 一些初始化代码可能很复杂，不适宜在每份测试代码里都出现；
3. 环境和测试业务可能分属业务方，初始化代码需要修改时不方便直接修改其它业务方代码。

因此，将这些环境初始化独立成通用逻辑是优化代码结构的一个选择。

### 提取初始化 Rule

Junit 提供了 TestRule 接口，可以控制测试代码执行前、执行后运行特定的片段，适宜这种初始化场景，那么我们将这个初始化抽成一个 Rule：

```kotlin
class InitRule : TestRule {
    override fun apply(base: Statement?, description: Description?): Statement {
        return object : Statement() {
            override fun evaluate() {
                print("Init to 10")
                InitData.init(10)
                base?.evaluate()
            }
        }
    }
}
```

那么测试代码就可以相应变成：

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [26])
class RuleTest {
    @Rule
    @JvmField
    val initRule = RuleChain.outerRule(InitRule())

    @Test
    fun testInit() {
        assertEquals(10, InitData.getData())
    }
}
```

### 使用 Powermock

在我们的项目中，经常用到了 Powermock 来对依赖项进行 mock 操作。大家都知道，Powermock 是使用定制的 Classloader 加载测试类，达到替换被依赖类的目的。那么在与 Rule 的配合使用中就出现了问题：

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [26])
class RuleTest {
    @Rule
    @JvmField
    val powerMockRule = PowerMockRule()
    @Rule
    @JvmField
    val initRule = RuleChain.outerRule(InitRule())

    @Test
    fun testInit() {
        // Test should fail here because initData has not been inited
        assertEquals(10, InitData.getData())
    }
}

class InitRule : TestRule {
    override fun apply(base: Statement?, description: Description?): Statement {
        return object : Statement() {
            override fun evaluate() {
                print("Init to 10")
                InitData.init(10)
                base?.evaluate()
            }
        }
    }
}
```

在运行这个测试时，将会失败：

> Init to 10
> java.lang.AssertionError:
> Expected :10
> Actual   :0

## 问题分析

根据输出，明明已经调用了初始化了，为什么在测试方法内还会说 `initData` 是默认值0呢？首先根据断点大法，看看测试执行过程中发生了什么。

### Rule 的执行顺序

在各处打上断点之后，会发现自定义的 `InitRule` 先被执行，再执行 `PowerMockRule`。似乎 Rule 的执行顺序对测试结果有影响。先查看 JUnit 里的 [`Rule`文档](https://junit.org/junit4/javadoc/4.12/org/junit/Rule.html) 对于顺序的描述：

> However, if there are multiple fields (or methods) they will be applied in an order that depends on your JVM's implementation of the reflection API, which is undefined, in general.

JUnit 本身不保证 Rule 的执行顺序，那么先关注一下 `PowerMockRule` 内部进行了什么操作。

### PowerMockRule 做了什么

先从表象上看，可以观察到一个现象： `InitRule` 执行时引用到了 `InitData` 类，那么其单例对象在类加载时就会被创建好，拥有一个内存地址，可以在 IDE 的 debug 模式中被看到；

继续运行，在测试方法体被执行时再次观察，会发现 `InitData` 的单例对象地址发生了改变，成为了一个新的对象，在没有其它调用初始化的地方时，其中存着的值当然就是默认值 0 了。

那么，在运行过程中，单例对象为什么会被替换呢？

从[上文提到的](#%e4%bd%bf%e7%94%a8-powermock) Powermock 的原理中其实能有一个猜测：因为 Powermock 使用了自己定制的 Classloader ，那么会不会是因为新的 Classloader 加载了单例类之后生成了新的单例对象，并且在测试执行时使用了新对象导致的？

观察断点命中时的调用栈信息，我们能发现一点端倪：

在 AbstractClassloaderExecutor#executeWithClassLoader 中， Powermock 将测试类深度复制（deepclone）了一次，并且使用一个新的 classloader 去加载，然后才再执行测试代码。这个新的 classloader 是由 `PowerMockRule` 中返回的 `PowerMockStatement` 初始化的，由 `MockClassLoaderFactory` 生成的定制版，用处不言而喻当然是为了实现各种 mock 功能了。

由一个新的 classloader 加载测试类之后，其依赖的 `InitData` 类需要由相同的 classloader 加载，才能被正确使用。在重新加载的过程中生成了新的单例对象，其中的变量都保持着未初始化的状态。

由此可以扩展开来想，对于 `PowerMockRule` 这种需要更换 classloader 的环境操作，可以认为执行前和执行后是完全不相干的两套运行环境，切换环境前运行的所有初始化行为都不会对切换后的环境产生影响。因此我们定制的 `InitRule` 需要在 `PowerMockRule` 之后执行。

但是上文也提到了，对于 field 级的 Rule ，执行顺序是不被保证的，所幸 JUnit 提供了 `RuleChain` 来控制执行顺序。

## 使用 RuleChain 再次优化

JUnit 提供了 [`RuleChain`](https://junit.org/junit4/javadoc/4.12/org/junit/rules/RuleChain.html) 的方式来让我们手动地控制各种 `Rule` 的执行顺序。假设我们为测试类声明了这样一套 Rule：

```kotlin
@Rule
@JvmField
val rule = RuleChain.outerRule(Rule1()).around(Rule2()).around(Rule3())
```

那么，测试执行时将会严格按照 Rule1 -> Rule2 -> Rule3 的顺序进入，即执行各个 Rule 中 `base.evaluate()` 之前的语句，然后执行 `@Test` 测试方法，再按照 Rule3 -> Rule2 -> Rule1 的顺序退出，即执行 `base.evaluate()` 之后的语句。

但是，`RuleChain` 所接受的 Rule 需要是 `TestRule` 的子类，而 `PowerMockRule` 所使用的是即将要被废弃的 `MethodRule` ，并不兼容，需要进行一个适配转换的操作。

### 使用 Adapter

除了等待 `PowerMockRule` 被正式升级为 `TestRule` 之外，可以使用其 [issue 列表](https://github.com/powermock/powermock/issues/396) 中提供的一个适配器，将 `MethodRule` 转换为 `TestRule` 实现：

```kotlin
class TestRuleAdapter(private val rule: MethodRule) : TestRule {
    override fun apply(base: Statement, description: Description): Statement {
        return rule.apply(base, createFrameworkMethod(description), getTestObject(description))
    }

    private fun createFrameworkMethod(description: Description): FrameworkMethod {
        try {
            val methodName = description.methodName
            val c = getTestClass(description)
            val m = c.getDeclaredMethod(methodName)
            return FrameworkMethod(m)
        } catch (e: Exception) {
            throw IllegalStateException(e)
        }
    }

    private fun getTestClass(description: Description): Class<*> {
        return description.testClass
    }

    private fun getTestObject(description: Description): Any {
        try {
            return getTestClass(description).newInstance()
        } catch (e: InstantiationException) {
            throw IllegalStateException(e)
        } catch (e: IllegalAccessException) {
            throw IllegalStateException(e)
        }
    }
}
```

然后我们就可以优雅地使用 `RuleChain` 管理我们的测试 Rule 了：

```kotlin
class AdapterRuleTest {
    @Rule
    @JvmField
    val initRule = RuleChain.outerRule(TestRuleAdapter(PowerMockRule())).around(InitRule())
    @Mock
    lateinit var mock: MockInterface

    @Before
    fun init() {
        MockitoAnnotations.initMocks(this)
        `when`(mock.makeInt()).thenReturn(100)
    }

    @Test
    fun testInit() {
        assertEquals(10, InitData.getData())

        //make sure powermock is working now
        assertEquals(100, mock.makeInt())
    }
}

interface MockInterface {
    fun makeInt(): Int
}
```

## 还有一个执行位置

除了使用 `Rule` `@Before` 初始化之外，JUnit 还提供了 `@BeforeClass` 这么一个注解来执行一些操作，我们来试试能不能使用：

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [26])
class BeforeClassTest {
    @Rule
    @JvmField
    val powerMockRule = PowerMockRule()

    companion object {
        @BeforeClass
        @JvmStatic
        fun init() {
            InitData.init(10)
        }
    }
    @Test
    fun testInit() {
        // Test should fail here because initData has not been inited
        assertEquals(10, InitData.getData())
    }
}
```

同样的，我们会得到一个断言错误：

> java.lang.AssertionError:
> Expected :10
> Actual   :0

可以得出结论， `@BeforeClass` 是在 Rule 之前被执行的。

## 结语

使用一个 Adapter 将 PowerMockRule 包装成 TestRule 之后，就能使用 RuleChain 完整地控制 Rule 执行顺序了，这样就可以简单地用上其他人包装好的 Rule ，简化自己的测试代码。

文中所写的示例代码已经上传到了[我的 Github 仓库](https://github.com/xiaozhikang0916/TestRuleSample)中，执行 `test` 将会在上文提到的错误使用的位置断言失败。

但是，这个 Adapter 只能使用在普通的测试中，[参数化测试](https://github.com/junit-team/junit4/wiki/Parameterized-tests)不能使用，需要后续再扩展。
