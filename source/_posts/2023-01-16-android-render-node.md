---
title: 在 Android 中使用 Render Node 加速渲染
date: 2023-01-16 16:47:04
tags:
    - Android
    - RenderNode
    - Canvas
---

在刚开始安卓开发的时候，前辈教导我：如果视图渲染出现了什么问题，就把硬件加速关掉。这确实让我绕开了不少显示上的问题。在后面的需求开发中，正好碰到了一个功能点：需要使用一个图片作为蒙层（称为蒙层图片），实际图片（称为内容图片）需要只在蒙层图片有不透明像素的地方显示，以实现对图片的特殊裁剪效果。

类似的功能已经做过很多次了，实现写起来也是信手拈来：

## 使用 Canvas 绘制蒙层

将两张图片绘制到一起，需要使用到的是 [`PorterDuff.Mode`](https://developer.android.com/reference/android/graphics/PorterDuff.Mode) 混合模式，其中根据蒙层和实际图片绘制顺序的不同，可以选择使用的有 `DST_IN` 、 `SRC_ATOP` 等；在绘制时，可以调用 `Paint.setXfermode` 为画笔设置不同的混合模式，实现混合效果。

在绘制蒙层和图片时，画布 `Canvas` 上可能已经有其他元素存在了，直接叠加蒙层的话会在已有的内容上也使用像素混合效果，产生与预期不服的结果。为了避免类似的问题，需要在绘制蒙层图片和内容图片之前，使用 `Canvas.saveLayer` 将绘制的元素临时保存到另一个图层中，实现与画布现有元素互不干扰的效果。

```kotlin
// 准备绘制资源
val head = ContextCompat.getDrawable(context, R.drawable.head)
val mask = ContextCompat.getDrawable(context, R.drawable.mask)

private val maskPaint = Paint().apply {
    // 先画蒙层、再画内容，使用 SRC_IN 模式
    xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)
}
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    // 保存原图图层，与画布上原有元素互不干扰
    val saveCount = canvas.saveLayer(maskRectF, null)
    mask?.draw(canvas)
    // 保存蒙层图层，在恢复时使用 maskPaint 中设置的混合模式进行像素混合
    val layerCount = canvas.saveLayer(rectF, maskPaint)
    head?.draw(canvas)

    // 恢复图层以应用像素混合
    canvas.restoreToCount(layerCount)
    canvas.restoreToCount(saveCount)
}
```

### 直接使用 Canvas 绘制蒙层的问题

上面的代码已经能实现需求功能了，但是 `Canvas.saveLayer` 上的注释让我耿耿于怀：

> This behaves the same as save(), but in addition it allocates and redirects drawing to an offscreen rendering target.
Note: this method is **very expensive**, incurring more than double rendering cost for contained content. Avoid using this method when possible and instead use a hardware layer on a View to apply an xfermode, color filter, or alpha, as it will perform much better than this method.

在这段节选的方法注释中可以得出几个结论：

* 这个方法的调用开销**十分昂贵**，应当尽量避免使用；
* 调用此方法后**将失去硬件加速效果**，无法从GPU中获益。

所以，我开始寻找使用其他途径实现这个功能的办法。

## 安卓的 RenderNode 加速

前不久正好看到了一篇安卓团队发布的，介绍[使用RenderNode进行模糊绘制](https://medium.com/androiddevelopers/rendernode-for-bigger-better-blurs-ced9f108c7e2)的博客文章，其中介绍了如何在自定义控件中使用 `RenderNode` 进行高级的、定制化的渲染显示效果。其中吸引我关注的有以下几点：

* 保持硬件加速特性，绘制性能优；
* 各种渲染效果可以自行组合叠加；
* 不同元素之间的效果和渲染不会冲突。

这些特点正好能满足我在使用 `Canvas.saveLayer` 时所遇到的问题，于是赶紧学习一波 `RenderNode` 的使用姿势，并实现一下上面的功能。

## RenderNode 的使用

如上博文所说， `RenderNode` 是在 API 29 及以上系统才能公开访问的类。使用时先调用 `RenderNode.setPosition` 设置绘制的位置和范围，然后使用 `RenderNode.recordingCanvas` 获取一个 `RecordingCanvas` ，该画布在使用方式上与普通画布别无二致，但是会把开发者在上面的操作命令记录下来，在实际绘制时再执行，达到加速的目的。最后，使用 `Canvas.drawRenderNode` 将记录好内容的 `RenderNode` 绘制到目标视图的 `onDraw` 中传入的画布中，将其渲染到屏幕上。

### 用 RenderNode 渲染蒙层

那么，就直接尝试一下使用 `RenderNode` 来实现同样的蒙层功能。

首先，为了避免在 `onDraw` 过程中重复创建新对象，我们应当在视图构建时就准备好需要的资源：

```kotlin
// 准备绘制资源
val head = ContextCompat.getDrawable(context, R.drawable.head)
val mask = ContextCompat.getDrawable(context, R.drawable.mask)

// 准备画笔，先画内容、再画蒙层，蒙层叠加在内容上，使用 DST_IN 模式
val paint = Paint().apply {
    blendMode = BlendMode.DST_IN
}

// 使用两个 RenderNode 分别绘制内容和蒙层，再将两个 RenderNode 混合
val headRender = RenderNode("head")
val headMaskRender = RenderNode("head_mask")
```

在绘制时，实现思路与 `Canvas` 类似，需要使用到 `RenderNode.setUseCompositingLayer` ，将绘制的内容另外开辟空间保存，在更新到屏幕上时才进行实际的渲染和混合。而不同的是， `RenderNode.setUseCompositingLayer` 会将内容保存在显存空间中，能提高渲染速度，并且不影响内存使用。

```kotlin
// 一个辅助函数
inline fun RenderNode.withRecording(block: (canvas: Canvas) -> Unit) {
    block(this.beginRecording())
    this.endRecording()
}

override fun onDraw(canvas: Canvas?) {
    super.onDraw(canvas)
    // 将两个render node设置初始位置和大小
    headRender.setPosition(0, 0, width, height)
    headMaskRender.setPosition(0, 0, width, height)
    
    // 设置分层绘制，并给蒙层设置paint
    headRender.setUseCompositingLayer(true, null)
    headMaskRender.setUseCompositingLayer(true, paint)
    
    // 记录蒙层内容
    headMaskRender.withRecording {
        mask?.draw(it)
    }

    // 内容实际绘制
    headRender.withRecording {
        // 先画内容
        head?.draw(it)
        // 再将蒙层叠加上来
        it.drawRenderNode(headMaskRender)
    }
    
    // 将混合结果输出到 view 中
    canvas?.drawRenderNode(headRender)
}
```

于是，我们就成功地使用 `RenderNode` 实现了蒙层混合的效果了。

## RenderNode 与 Canvas 的性能对比

使用 `RenderNode` 进行蒙层渲染，实际收益能有多大？既然两种渲染方式都实现了，可以直接实现一个Demo进行对比，源代码将在文末贴上。

### 温和场景的对比

首先模拟一个在普通场景下的性能表现：使用 `RecyclerView` 的纵向布局，模拟带有头像的内容列表中的表现，使用 `adb shell dumpsys gfxinfo` 获取demo应用的绘制信息。

> Stats since: 362520548662ns
>
> Total frames rendered: 1393
>
> Janky frames: 17 (1.22%)
>
> 50th percentile: 5ms
>
> 90th percentile: 5ms
>
> 95th percentile: 5ms
>
> 99th percentile: 11ms
>
> Number Missed Vsync: 5
>
> Number High input latency: 185
>
> Number Slow UI thread: 9
>
> Number Slow bitmap uploads: 0
>
> Number Slow issue draw commands: 6
>
> Number Frame deadline missed: 11
>
> 50th gpu percentile: 1ms
>
> 90th gpu percentile: 2ms
>
> 95th gpu percentile: 3ms
>
> 99th gpu percentile: 12ms

同样的内容，如果使用 `Canvas` 进行绘制，输出如下

> Stats since: 362520548662ns
>
> Total frames rendered: 2244
>
> Janky frames: 69 (3.07%)
>
> 50th percentile: 5ms
>
> 90th percentile: 5ms
>
> 95th percentile: 5ms
>
> 99th percentile: 18ms
>
> Number Missed Vsync: 7
>
> Number High input latency: 279
>
> Number Slow UI thread: 12
>
> Number Slow bitmap uploads: 0
>
> Number Slow issue draw commands: 17
>
> Number Frame deadline missed: 25
>
> 50th gpu percentile: 1ms
>
> 90th gpu percentile: 2ms
>
> 95th gpu percentile: 6ms
>
> 99th gpu percentile: 12ms

可以看出，使用 `RenderNode` 渲染的界面，在卡顿率上比 `Canvas` 有了明显的改善（ 3.07% -> 1.22%），99分位的绘制时间也明显变少，可以认为使用 `RenderNode` 能降低这种工况下画面卡顿的概率。

如果绘制出页面绘制所耗时间的直方图，也能看出明显区别：

![RenderNode 一列](https://i0.hdslb.com/bfs/new_dyn/bc9c71ac92fc2cb380ef40dcab9460761564902300.png)

![Canvas 一列](https://i0.hdslb.com/bfs/new_dyn/ccd9341cf8153c4074b56c1fc83a552b1564902300.png)

可以看出，使用 `RenderNode` 的渲染时间已经明显比 `Canvas` 减少，单纯从平均总耗时来看，平均每帧绘制时间从 4.53ms 降低为 2.49ms，降幅达45%。

### 极端场景的比对

将demo稍作修改，可以得到一个极端场景：使用 `GridLayoutManager`，在页面中显示每行4个的图像，将屏幕中出现的元素数量最大化，能更明显地看出双方的性能对比。

首先是 `RenderNode` 选手的数据：

> Stats since: 1675600047954ns
>
> Total frames rendered: 490
>
> Janky frames: 9 (1.84%)
>
> 50th percentile: 5ms
>
> 90th percentile: 5ms
>
> 95th percentile: 6ms
>
> 99th percentile: 24ms
>
> Number Missed Vsync: 2
>
> Number High input latency: 234
>
> Number Slow UI thread: 4
>
> Number Slow bitmap uploads: 0
>
> Number Slow issue draw commands: 4
>
> Number Frame deadline missed: 5
>
> 50th gpu percentile: 2ms
>
> 90th gpu percentile: 7ms
>
> 95th gpu percentile: 7ms
>
> 99th gpu percentile: 7ms

虽然同屏元素多了3倍，但此时的绘制性能没有受到太多影响；而另一方面 `Canvas` 选手的表现：

> Stats since: 1675600047954ns
>
> Total frames rendered: 912
>
> Janky frames: 425 (46.60%)
>
> 50th percentile: 6ms
>
> 90th percentile: 24ms
>
> 95th percentile: 25ms
>
> 99th percentile: 34ms
>
> Number Missed Vsync: 129
>
> Number High input latency: 439
>
> Number Slow UI thread: 93
>
> Number Slow bitmap uploads: 0
>
> Number Slow issue draw commands: 218
>
> Number Frame deadline missed: 220
>
> 50th gpu percentile: 2ms
>
> 90th gpu percentile: 6ms
>
> 95th gpu percentile: 7ms
>
> 99th gpu percentile: 7ms

可以看到， `Canvas` 在这种极端复杂的场景下性能表现退步极大，卡顿帧比例达到了46.60%，几乎是不可用的状态。

再看双方的绘制时间直方图：

![RenderNode 四列](https://i0.hdslb.com/bfs/new_dyn/6bf08fa040a60fe795c71f3cc03d984a1564902300.png)

![Canvas 四列](https://i0.hdslb.com/bfs/new_dyn/f9f27839d0516764a456b87485b18a071564902300.png)

在平均绘制时间上， `RenderNode` 仍然保持了2.81ms的优秀绘制时间，没有对用户造成影响；而 `Canvas` 在此种工况下的平均绘制时间来到了21.30ms，差出了一个数量级，使得应用无法保持60fps渲染效率。

## 总结

通过一个简单的demo可以看出，使用 `RenderNode` 进行复杂特性的渲染确实能保持相对较高的渲染效率，能提升用户体验。但它并不是万能的，首先 API level 29 的要求相信就已经能拦住绝大多数生产项目的使用，在不提升项目版本的情况下，只能为同一个功能点写两份代码，保留 `Canvas` 的实现以运行在低版本的手机上。

但，对我来说是探索安卓硬件加速渲染的第一步，是从 `把硬件加速关掉` 到主动从硬件加速中受益的转变。

以上实例项目已开源，[项目地址](https://github.com/xiaozhikang0916/RenderNode)。
