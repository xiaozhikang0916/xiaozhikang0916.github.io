---
title: java 构造方法的重载
date: 2020-03-22 10:37:35
tags:
    - Java
    - Constructor
---

在安卓项目中经常有需要自定义 `View` 控件的场景。最近发现了两种声明构造方法的风格：

```java
class View1 extends View {
     public View1(Context context) {
        this(context, null);
    }

    public View1(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public View1(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, 0);
        init();
    }

    private void init() {
        //do some init works
    }
}

class View2 extends View {
     public View1(Context context) {
        super(context);
        init();
    }

    public View1(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public View1(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, 0);
        init();
    }

    private void init() {
        //do some init works
    }
}
```

一开始我会认为类似于 `View2` 的写法是错误的，因为在调用一个参数的构造方法时，调用了超类的构造方法，其超类的一个参数构造方法则委托给两个参数的构造方法实现，就会调用到子类的两个参数的构造方法，以此类推，最后会调用3次 `init()` 方法。

但是看见了多次这样的写法后，不免对自己的看法产生了怀疑，决定看一看具体的表现。

## 构造方法的重载

首先实验一下，类似于 `View2` 的写法会不会真的出现问题：

```java
public class Parent {
    public Parent(int i) {
        this(i, 0);
    }

    public Parent(int i, int n) {
        System.out.println(String.format("Call constructor of parent with %d, %d", i, n));
    }
}

public class Child extends Parent {

    public Child(int i) {
        super(i);
        print("Call constructor 1 of child");
    }

    public Child(int i, int n) {
        super(i, n);
        print("Call constructor 2 of child");
    }

    public void print(String s) {
        System.out.println(s);
    }
}

public static void main(String... args) {
    final Child c = new Child(1);
}
```

得到输出：

> Call constructor of parent with 1, 0
>
> Call constructor 1 of child

可见，子类的构造方法2并没有 `重载` 或者说 `复写` 超类的构造方法2。其原因还得到字节码中一探究竟。

## Java 中不同的 invoke

通过反编译 `Parent.class` 查看超类的构造方法字节码，在其中发现了感兴趣的点：

```bytecode
 public <init>(I)V
   L0
    LINENUMBER 10 L0
    ALOAD 0
    ILOAD 1
    ICONST_0
    INVOKESPECIAL com/bilibili/test/Parent.<init> (II)V
   L1
    LINENUMBER 11 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/bilibili/test/Parent; L0 L2 0
    LOCALVARIABLE i I L0 L2 1
    MAXSTACK = 3
    MAXLOCALS = 2
```

其中，在构造方法1调用构造方法2的语句里，其字节码使用的是 `INVOKESPECIAL` ，而普通的 `public` 或者 `procted` 方法则是使用 `INVOKEVIRTUAL` 来调用。根据 oracle [对两者的解释](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokespecial)：

> The difference between the invokespecial instruction and the invokevirtual instruction (§invokevirtual) is that invokevirtual invokes a method based on the class of the object. The invokespecial instruction is used to invoke instance initialization methods (§2.9) as well as private methods and methods of a superclass of the current class.

具体地说，`INVOKESPECIAL` 是专门用来调用 超类 、 构造 和 私有 方法的，其具体调用的方法入口与调用的声明类有关，跟运行时实例的类型关系不大；而 `INVOKEVIRTUAL` 调用的实际入口则是跟运行时实例的类型相关，因此才能实现多态与方法复写。

而 `INVOKESPECIAL` 所适用的场景也是有其意义在的：

1. 调用超类方法，既然已经明确了调用超类，那么就不应该调用到子类的复写方法，否则调用链就变成了递归；
2. 调用构造方法，特别是在同类型的不同参数的构造方法的委托里，通过此指令明确委托调用的是同类的构造方法，保证超类确实是在子类构造实际执行之前构造完成；
3. 调用私有方法，用此指令明确调用的是自己类型的私有方法，保证了私有方法的不可复写性。

在实现类的继承时，私有方法在子类中不可见、不可复写的特性比较被人熟知。
但构造方法有类似的特性，调用 `this(...)` 时不会被子类“覆盖”的特性需要自己特别注意了。
