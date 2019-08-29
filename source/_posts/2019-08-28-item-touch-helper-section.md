---
title: ItemTouchHelper实现拖动分组与定制
date: 2019-08-28 17:52:55
tags:
    - Android
    - ItemTouchHelper
    - RecyclerView
---

Android 在 RecyclerView 中使用了 [ItemTouchHelper](https://developer.android.com/reference/android/support/v7/widget/helper/ItemTouchHelper) 来支持列表项的移动、横扫功能。
最近在项目中用到了这部分功能，并且有限定拖动触发区域、拖动范围限制的要求，在此做一点记录。

## 基础使用

要为 `RecyclerView` 用上 `ItemTouchHelper` ，需要自己实例化一个 `ItemTouchHelper` 并传入一个自定义的  `ItemTouchHelper.Callback`。

```kotlin
val itemTouchHelper = ItemTouchHelper(
    object : ItemTouchHelper.Callback() {
        ...
    }
)

itemTouchHelper.attachToRecyclerView(recyclerView)
```

自己所需要的移动、横扫功能定制就由传入的 `Callback` 来定制。

在传入的 `Callback` 中，由3个需要实现的抽象方法：

```kotlin
abstract fun getMovementFlags(recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder): Int
```

这个方法用来判定一个 `ViewHolder` 支持什么样的方式移动，一般直接调用`makeMovementFlags(int fragFlags, int swipeFlags)` 方法，来构造这个 flag int ，如

```kotlin
return makeMovementFlags(ItemTouchHelper.UP or ItemTouchHelper.DOWN, 0)
```

表示所有的项目都支持往上和往下方向的拖动，不支持横扫手势。

与拖动、横扫手势对应的，有两个事件回调：

```kotlin
abstract fun onMove(
    recyclerView: RecyclerView,
    viewHolder: RecyclerView.ViewHolder,
    target: RecyclerView.ViewHolder
): Boolean

abstract fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int)
```

其中 `onMove` 在列表项被移动到一个新的位置上时被调用，我们在此处理数据交换的流程，如交换数据集中的位置、通知 `adapter` 更新；

`onSwipe` 在触发横扫手势时被调用，用来更新界面以显示横扫后需要出现的操作按钮。

## 定制

在我的业务中，需要限制拖动触发区域，在手点在拖动锚点上时触发拖动操作，其余区域即使长按也不能拖动；列表在视觉上分为两部分，列表项不能在组间互相交换位置，只能在组内交换，因此需要有更深的定制操作。

### 修改拖动触发时机

`ItemTouchHelper` 的默认行为是长按列表项就触发拖动操作，我们需要修改触发时机，首先是禁用长按，只需要重载 `Callback` 的一个方法：

```kotlin
object : ItemTouchHelper.Callback() {
    override fun isLongPressDragEnabled(): Boolean = false
}
```

然后为列表项中的 `view` 绑定点击事件，通过主动调用 `itemTouchHelper.startDrag(viewHolder)` 触发拖动事件：

```kotlin
dragAnchor.setOnTouchListener { _, event ->
    when(event.action) {
        MotionEvent.ACTION_DOWN -> {
            itemTouchHelper.startDrag(this@ViewHolder)
        }
    }
}
```

### 拖动分组

在默认行为里，列表项可以拖动到任意其它列表项上，我们需要重载 `Callback.chooseDragTarget` ，实现拖动范围的限制。

首先观察这个方法的签名：

```java
public ViewHolder chooseDropTarget(@NonNull ViewHolder selected, @NonNull List<ViewHolder> dropTargets, int curX, int curY)
```

我们通过参数可以拿到当前被拖动的列表项，
以及可以被选为释放目标的其它列表项 `dropTargets` ，
而默认实现则是从 `dropTargets` 这个列表中选择最合适的释放位置。
那么我们就可以通过过滤不合适的释放位置，达到限制拖动的目的：

```kotlin
override fun chooseDropTarget(
    selected: RecyclerView.ViewHolder,
    dropTargets: MutableList<RecyclerView.ViewHolder>,
    curX: Int,
    curY: Int
): RecyclerView.ViewHolder? {
    val filtered = dropTargets.filter { isSameSection(target, it) }
    return super.chooseDropTarget(selected, filtered, curX, curY)
}
```

> 在使用kotlin时，切记要在重载方法时，将返回类型改为可空 `RecyclerView.ViewHolder?` ，以节约一次编译时间。

### 拖动阴影

在 `RecyclerView` 中，列表项的布局和绘制顺序一般来说是从上到下的，如果不加任何限制，可能会出现的情况是：
将某一项往下拖动时，会因为下面的视图绘制顺序比被拖动项更后，使得被拖动项被其它视图覆盖，不符合一般直观感受。

`ItemTouchHelper` 对此的解决方案分为两种。在系统小于 `Lolipop 21` 的机子上，通过一个 `RecyclerView.ChildDrawingOrderCallback` 来更改绘制顺序，
保证被拖动项最后绘制，以免被其它视图覆盖。

在大于 21 的机子上，系统支持通过 [elevation](https://material.io/design/environment/elevation.html) 来控制视图的z轴前后关系，并实现阴影效果：

```java
class ItemTouchUIUtilImpl implements ItemTouchUIUtil {
    static final ItemTouchUIUtil INSTANCE =  new ItemTouchUIUtilImpl();

    @Override
    public void onDraw(Canvas c, RecyclerView recyclerView, View view, float dX, float dY,
            int actionState, boolean isCurrentlyActive) {
        if (Build.VERSION.SDK_INT >= 21) {
            if (isCurrentlyActive) {
                Object originalElevation = view.getTag(R.id.item_touch_helper_previous_elevation);
                if (originalElevation == null) {
                    originalElevation = ViewCompat.getElevation(view);
                    float newElevation = 1f + findMaxElevation(recyclerView, view);
                    ViewCompat.setElevation(view, newElevation);
                    view.setTag(R.id.item_touch_helper_previous_elevation, originalElevation);
                }
            }
        }

        view.setTranslationX(dX);
        view.setTranslationY(dY);
    }
    ...
}
```

`ItemTouchUIUtilImpl` 在大于21的机子上，会找到当前 `recyclerView` 中 **除被拖动项以外** `elevation` 的最大值，
并将最大值 **+1 px** 设置给被拖动项，以达到被拖动项在所有项目之上的效果，并且在手势完成后恢复为原来的值。这里有两点需要注意的：

1. 拖动时的 `elevation` 的差值固定为 1px，没有提供定制；
2. 拖动时的 `elevation` 值与自己的原值无关，因此不能通过拖动开始时修改 `elevation` 来达到定制效果；

因为上述的 2 ，如果我们需要定制拖动时的阴影效果，我们需要另外一种方式来控制z轴的前后关系：
>z = elevation + translationZ

我们可以在拖动前后修改视图的 `translationZ` 值：

```kotlin
override fun onSelectedChanged(
    viewHolder: RecyclerView.ViewHolder?,
    actionState: Int
) {
    super.onSelectedChanged(viewHolder, actionState)
    if (actionState == ItemTouchHelper.ACTION_STATE_DRAG) {
        viewHolder?.itemView?.let {
            // add 1 dp to final z
            ViewCompat.setTranslationZ(it, 3f)
        }
    }
}

override fun clearView(
    recyclerView: RecyclerView,
    viewHolder: RecyclerView.ViewHolder
) {
    super.clearView(recyclerView, viewHolder)
    ViewCompat.setTranslationZ(viewHolder.itemView, 0f)
}
```

这样，对拖动效果的定制就完成了。

## 后记

为了验证上述关于 `elevation` 的两个结论，我们可以做一点好玩的事情：

### elevation 值固定

拖动时的 `elevation` 的差值固定为 1px，没有提供定制。如果我们尝试修改被拖动项的 `elevation` ，将没有效果。

```kotlin
override fun onSelectedChanged(viewHolder: RecyclerView.ViewHolder?, actionState: Int) {
    super.onSelectedChanged(viewHolder, actionState)
    if (actionState == ItemTouchHelper.ACTION_STATE_DRAG) {
        viewHolder?.itemView?. let {
            //无效
            ViewCompat.setElevation(it, 3f)
        }
    }
}
```

### elevation 值与自己无关

因为被拖动项的 `elevation` 值与自己的原值无关，而是当前列表的最大值 +1 px，可以通过触发拖动时随意修改一个项的 `elevation` 来验证：

```kotlin
override fun onSelectedChanged(viewHolder: RecyclerView.ViewHolder?, actionState: Int) {
    super.onSelectedChanged(viewHolder, actionState)
    if (actionState == ItemTouchHelper.ACTION_STATE_DRAG) {
        // set the first child's elevation to 20 when some item is being dragged
        (0 until recyclerView.childCount).asSequence().map {
            recyclerView.getChildAt(it)
        }.filter { it is LinearLayout }.first().apply {
            ViewCompat.setElevation(this, 10f)
        }
        viewHolder?.itemView?. let {
            ViewCompat.setElevation(it, 20f)
        }
    }
}

override fun clearView(
    recyclerView: RecyclerView,
    viewHolder: RecyclerView.ViewHolder
) {
    super.clearView(recyclerView, viewHolder)
    (0 until recyclerView.childCount).asSequence().map {
        recyclerView.getChildAt(it)
    }.filter { it is LinearLayout }.first().apply {
        ViewCompat.setElevation(this, 0f)
    }
    viewHolder.itemView. let {
        ViewCompat.setElevation(it, 0f)
    }
}
```

观察到的现象：

* 如果被拖动的是列表第一项，他的z会变成 1px；
* 如果被拖动的不是第一项，那么第一项的z会变成 10px，被拖动的z变成 11px。
