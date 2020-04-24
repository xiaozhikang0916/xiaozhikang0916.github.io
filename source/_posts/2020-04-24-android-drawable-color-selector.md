---
title: Android xml 中 drawable、color 的混用与 selector
date: 2020-04-24 08:23:03
tags:
    - Android
    - xml
    - layout
    - drawable
    - color
    - selector
---

平时在安卓项目中需要编写 xml 布局时，我都习惯需要接收 `drawable` 资源的属性中（ 如 `android:background` ）可以按需传入 `drawable` 资源 （ 如 `@drawable/layout_bg` ） 和 `color` 资源 （ 如 `@color/layout_bg_color`），并且默认在 xml 资源中 color 资源是可以代替 drawable 使用的。

但是最近在布局中使用 `selector` 时，发现表现并不如自己所料。

## 问题

需求是在一排 tab 中，被选中的 tab 使用一种背景色和文字颜色，未被选中的其它所有 tab 都使用另一种背景色和文字颜色。在这样的需求下，使用 `selector` ，根据 `state_selected` 属性来区分颜色就很方便了。

color/tab_color.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="#777777" android:state_selected="true" />
    <item android:color="#555555" />
</selector>
```

color/tab_color.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="#555555" android:state_selected="true" />
    <item android:color="#777777" />
</selector>
```

layout/tab.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/tab_bg"
    android:padding="14dp">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textColor="@color/tab_color"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>
```

在代码中使用这个布局，并且设置好点击事件后，运行代码，却得到错误：

```
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.example.myapplication/com.example.myapplication.MainActivity}: android.view.InflateException: Binary XML file line #2: Binary XML file line #2: Error inflating class android.support.constraint.ConstraintLayout
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2778)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2856)
        at android.app.ActivityThread.-wrap11(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1589)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6494)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)
     Caused by: android.view.InflateException: Binary XML file line #2: Binary XML file line #2: Error inflating class android.support.constraint.ConstraintLayout
     Caused by: android.view.InflateException: Binary XML file line #2: Error inflating class android.support.constraint.ConstraintLayout
     Caused by: java.lang.reflect.InvocationTargetException
        at java.lang.reflect.Constructor.newInstance0(Native Method)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:334)
        at android.view.LayoutInflater.createView(LayoutInflater.java:647)
        at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:790)
        at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:730)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:492)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:423)
        ...
    Caused by: android.content.res.Resources$NotFoundException: Drawable com.example.myapplication:color/tab_bg with resource ID #0x7f040055
     Caused by: android.content.res.Resources$NotFoundException: File res/color/tab_bg.xml from drawable resource ID #0x7f040055
        at android.content.res.ResourcesImpl.loadDrawableForCookie(ResourcesImpl.java:820)
        at android.content.res.ResourcesImpl.loadDrawable(ResourcesImpl.java:630)
        at android.content.res.Resources.loadDrawable(Resources.java:886)
        at android.content.res.TypedArray.getDrawableForDensity(TypedArray.java:953)
        at android.content.res.TypedArray.getDrawable(TypedArray.java:928)
        ...
    Caused by: org.xmlpull.v1.XmlPullParserException: Binary XML file line #3: <item> tag requires a 'drawable' attribute or child tag defining a drawable
        at android.graphics.drawable.StateListDrawable.inflateChildElements(StateListDrawable.java:189)
        at android.graphics.drawable.StateListDrawable.inflate(StateListDrawable.java:122)
        at android.graphics.drawable.DrawableInflater.inflateFromXmlForDensity(DrawableInflater.java:142)
```

## 错误分析

在异常信息中，可以得知 tab 的根布局就渲染失败了，其原因是为其设置的 background 资源 `@color/tab_bg` 解析失败，需要为其设置 `drawable` 属性，或者需要有定义了 drawable 的子 tag。

进入源码中仔细查看，一个 `selector` 资源对应会生成一个 `StateListDrawable` ，其中的 `inflateChildElements` 方法用来解析 xml 中的每一个 `item` ，其源码如下：

```java
private void inflateChildElements(Resources r, XmlPullParser parser, AttributeSet attrs,
        Theme theme) throws XmlPullParserException, IOException {
    final StateListState state = mStateListState;
    final int innerDepth = parser.getDepth() + 1;
    int type;
    int depth;
    while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
            && ((depth = parser.getDepth()) >= innerDepth
            || type != XmlPullParser.END_TAG)) {
        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        if (depth > innerDepth || !parser.getName().equals("item")) {
            continue;
        }

        // This allows state list drawable item elements to be themed at
        // inflation time but does NOT make them work for Zygote preload.
        final TypedArray a = obtainAttributes(r, theme, attrs,
                R.styleable.StateListDrawableItem);
        Drawable dr = a.getDrawable(R.styleable.StateListDrawableItem_drawable);
        a.recycle();

        final int[] states = extractStateSet(attrs);

        // Loading child elements modifies the state of the AttributeSet's
        // underlying parser, so it needs to happen after obtaining
        // attributes and extracting states.
        if (dr == null) {
            while ((type = parser.next()) == XmlPullParser.TEXT) {
            }
            if (type != XmlPullParser.START_TAG) {
                throw new XmlPullParserException(
                        parser.getPositionDescription()
                                + ": <item> tag requires a 'drawable' attribute or "
                                + "child tag defining a drawable");
            }
            dr = Drawable.createFromXmlInner(r, parser, attrs, theme);
        }

        state.addStateSet(states, dr);
    }
}
```

`parser` 传入的时候，便已经处于第一个 `selector` 的起始标记之后，经过 14 行的的循环语句后， `parser` 位于第一个 `item` 的起始标记之前，22 行里尝试从 `item` 项中获取 `drawable` 属性，以其中的值获取一个 `Drawable` 资源。

如果上述获取资源失败，在 30 行就会进入内部解析流程，将 `parser` 定位到内部第一个起始标记之前，然后把这个 `parser` 传递给 `Drawable.createFromXmlInner` ，解析出一个合适的 `drawable` 作为当前状态的图片资源。

在我刚刚所写的 `selector` 资源中，我简单地认为作为背景图片资源的 selector 也可以跟颜色资源一样，只为其写了 `android:color` 属性，恰好不满足源码中对于资源定义的要求，解析失败就是必然的了。

## 普通 color 可以替代 drawable 的原因

在知道了作为 `color` 的 `selector` 资源不能代替 `drawable` 后，我决定再探究一下为什么普通的颜色资源可以代替图片使用。其解析入口正是上面分析中提到的 `Drawable.createFromXmlInner` ，而内部调用到最深处使用的是 `android.graphics.drawable.DrawableInflater#inflateFromXmlForDensity` ：

```java
@NonNull
Drawable inflateFromXmlForDensity(@NonNull String name, @NonNull XmlPullParser parser,
        @NonNull AttributeSet attrs, int density, @Nullable Theme theme)
        throws XmlPullParserException, IOException {
    // Inner classes must be referenced as Outer$Inner, but XML tag names
    // can't contain $, so the <drawable> tag allows developers to specify
    // the class in an attribute. We'll still run it through inflateFromTag
    // to stay consistent with how LayoutInflater works.
    if (name.equals("drawable")) {
        name = attrs.getAttributeValue(null, "class");
        if (name == null) {
            throw new InflateException("<drawable> tag must specify class attribute");
        }
    }

    Drawable drawable = inflateFromTag(name);
    if (drawable == null) {
        drawable = inflateFromClass(name);
    }
    drawable.setSrcDensityOverride(density);
    drawable.inflate(mRes, parser, attrs, theme);
    return drawable;
}

@NonNull
@SuppressWarnings("deprecation")
private Drawable inflateFromTag(@NonNull String name) {
    switch (name) {
        case "selector":
            return new StateListDrawable();
        case "animated-selector":
            return new AnimatedStateListDrawable();
        case "level-list":
            return new LevelListDrawable();
        case "layer-list":
            return new LayerDrawable();
        case "transition":
            return new TransitionDrawable();
        case "ripple":
            return new RippleDrawable();
        case "adaptive-icon":
            return new AdaptiveIconDrawable();
        case "color":
            return new ColorDrawable();
        case "shape":
            return new GradientDrawable();
        case "vector":
            return new VectorDrawable();
        case "animated-vector":
            return new AnimatedVectorDrawable();
        case "scale":
            return new ScaleDrawable();
        case "clip":
            return new ClipDrawable();
        case "rotate":
            return new RotateDrawable();
        case "animated-rotate":
            return new AnimatedRotateDrawable();
        case "animation-list":
            return new AnimationDrawable();
        case "inset":
            return new InsetDrawable();
        case "bitmap":
            return new BitmapDrawable();
        case "nine-patch":
            return new NinePatchDrawable();
        case "animated-image":
            return new AnimatedImageDrawable();
        default:
            return null;
    }
}
```

这里就很简单了，识别到 tag 是 `color` 之后，系统自动创建一个 `ColorDrawable` ，相当于一个方便我们使用的系统适配；而 `selector` 对应的 `StateListDrawable` 的内部处理中，没有做类似的适配；对应于 color 中的 selector ，系统为我们创建的其实是 `ColorStateList` ， 并不属于一种 `Drawable` 。在这种情况下，就不能想当然地认为颜色可以作为图片使用了。

## 问题修复

知道错误原因之后修改就方便了，只需要按照其要求的规范写 `selector` 就好：

drawable/tab_bg.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_selected="true">
        <color android:color="#777777" />
    </item>
    <item>
        <color android:color="#555555" />
    </item>
</selector>
```

只是失去了普通颜色资源的既可用在 color 属性中，又可用在 drawable 属性中的便利性了。

又或者写出一种会被喷代码质量差的资源：

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_selected="true" android:color="#777777">
        <color android:color="#777777" />
    </item>
    <item android:color="#555555">
        <color android:color="#555555" />
    </item>
</selector>
```
