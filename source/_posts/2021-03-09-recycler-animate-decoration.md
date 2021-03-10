---
title: RecyclerView 中为 ItemDecoration 应用动画
tags:
  - Android
  - RecyclerView
  - ItemDecoration
  - Animation
date: 2021-03-09 23:06:49
---


在 `RecyclerView` 中，使用 `ItemDecoration` 来添加分割线或者侧边装饰条是十分常用的操作，可以实现分割线、装饰条的实现与列表元素解耦，和跨列表元素之间的装饰交互。

但是， `ItemDecoration` 的画面显示实现是需要开发者自己覆盖 `onDraw` 方法，在卡片之外自行将画面元素绘制上去。因其绘制过程与卡片本身没有关系，可以想象到的是，卡片在动画过程中不会自动地影响到 `ItemDecoration` 中所绘制的元素，需要开发者进行额外的适配工作。

## 贯穿分割线的绘制

`ItemDecoration` 最常见的一个用法是用来绘制卡片之间的贯穿型分割线。实现方法也非常简单，在需要绘制分割线的两张卡片之间，使用 `getItemOffsets` 方法留出足够的空间， `onDraw` 时在前一个卡片的 `bottom` 和后一个卡片的 `top` 之间绘制分割线即可。

```kotlin
    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        outRect.bottom = DIVIDER_HEIGHT
    }
    
    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)
        (0 until parent.childCount - 1).forEach { it ->
            val rect = RectF(
                0f,
                it.bottom.toFloat(),
                parent.width,
                it.bottom.toFloat() + DIVIDER_HEIGHT
            )
            c.drawRect(rect, DIVIDER_PAINT)
        }
    }
```

此实现在静态展现时还尚可，但是如果 `RecyclerView` 使用了默认的元素动画，在元素添加、删除等有位移动画时，分割线并不会随着动画一起移动，而是提前直接在位移结束位置上绘制出来，等着卡片元素到位。

### RecyclerView 元素动画的实现

为了解决这个问题，首先需要知道元素动画是如何实现的。

在使用默认动画的情况下，添加、删除一些卡片后，其余没有发生变化，但是需要移动位置的卡片是通过 `SimpleItemAnimator` 里的 `animatePersistence` 进行的：

```Java
    @Override
    public boolean animatePersistence(@NonNull RecyclerView.ViewHolder viewHolder,
            @NonNull ItemHolderInfo preInfo, @NonNull ItemHolderInfo postInfo) {
        if (preInfo.left != postInfo.left || preInfo.top != postInfo.top) {
            if (DEBUG) {
                Log.d(TAG, "PERSISTENT: " + viewHolder
                        + " with view " + viewHolder.itemView);
            }
            return animateMove(viewHolder,
                    preInfo.left, preInfo.top, postInfo.left, postInfo.top);
        }
        dispatchMoveFinished(viewHolder);
        return false;
    }
```

其中调用的是 `DefaultItemAnimator` 里的 `animateMove` 方法，最终由 `animateMoveImpl` 执行。

```Java

    @Override
    public boolean animateMove(final RecyclerView.ViewHolder holder, int fromX, int fromY,
            int toX, int toY) {
        final View view = holder.itemView;
        fromX += (int) holder.itemView.getTranslationX();
        fromY += (int) holder.itemView.getTranslationY();
        resetAnimation(holder);
        int deltaX = toX - fromX;
        int deltaY = toY - fromY;
        if (deltaX == 0 && deltaY == 0) {
            dispatchMoveFinished(holder);
            return false;
        }
        if (deltaX != 0) {
            view.setTranslationX(-deltaX);
        }
        if (deltaY != 0) {
            view.setTranslationY(-deltaY);
        }
        mPendingMoves.add(new MoveInfo(holder, fromX, fromY, toX, toY));
        return true;
    }

    void animateMoveImpl(final RecyclerView.ViewHolder holder, int fromX, int fromY, int toX, int toY) {
        final View view = holder.itemView;
        final int deltaX = toX - fromX;
        final int deltaY = toY - fromY;
        if (deltaX != 0) {
            view.animate().translationX(0);
        }
        if (deltaY != 0) {
            view.animate().translationY(0);
        }
        // TODO: make EndActions end listeners instead, since end actions aren't called when
        // vpas are canceled (and can't end them. why?)
        // need listener functionality in VPACompat for this. Ick.
        final ViewPropertyAnimator animation = view.animate();
        mMoveAnimations.add(holder);
        animation.setDuration(getMoveDuration()).setListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animator) {
                dispatchMoveStarting(holder);
            }

            @Override
            public void onAnimationCancel(Animator animator) {
                if (deltaX != 0) {
                    view.setTranslationX(0);
                }
                if (deltaY != 0) {
                    view.setTranslationY(0);
                }
            }

            @Override
            public void onAnimationEnd(Animator animator) {
                animation.setListener(null);
                dispatchMoveFinished(holder);
                mMoveAnimations.remove(holder);
                dispatchFinishedWhenDone();
            }
        }).start();
    }
```

可以看出来，位移动画的实现是通过修改其 `translationX` `translationY` 属性执行的。修改 `translation` 实现位移也是动画实现的常用方案，那么只要在绘制分割线的时候将 `translation` 属性也纳入计算即可。

```kotlin
    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        outRect.bottom = DIVIDER_HEIGHT
    }
    
    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)
        (0 until parent.childCount - 1).forEach { it ->
            val rect = RectF(
                0f,
                it.bottom.toFloat() + it.translationY,
                parent.width,
                it.bottom.toFloat() + DIVIDER_HEIGHT + it.translationY
            )
            c.drawRect(rect, DIVIDER_PAINT)
        }
    }
```

此时，分割线便能在列表元素进行位移动画过程中跟随一起移动了。

## 侧边装饰条的实现

在项目中，使用 `ItemDecoration` 实现列表的侧边装饰条也是很常见的操作。例如在卡片侧面绘制一条竖线，用来显示时间线效果。在接下来的讨论中，我们简单考虑这样一个需求：

* 数据结构中带有颜色字段，需要在卡片左侧绘制对应颜色的方块；
* 方块的绘制需要使用 `ItemDecoration` 实现；
* 方块需要跟随卡片进行位移、显隐动画。

### 简单实现

基于上一环节的成果，很快就能写出这样一段代码：

```kotlin
data class Data(
    val content: String,
    @ColorInt val color: Int
) {
    override fun toString(): String {
        return "Data(content=$content, color=#${String.format("%06x", color)}"
    }
}

class SideDecoration(context: Context, private val dataGetter: ((Int) -> Data?)) : RecyclerView.ItemDecoration() {
    private val paint = Paint()
    private val padding = context.resources.getDimensionPixelSize(R.dimen.item_side_padding)

    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)
        parent.children.map {
            it to parent.getChildAdapterPosition(it)
        }.map { (view, index) ->
            val data = dataGetter(index)
            Log.d("SideDecoration", "getting data to draw of index $index")
            view to data
        }.map { (view, data) ->
            RectF(
                0f,
                view.top + view.translationY,
                padding.toFloat(),
                view.bottom + view.translationY
            ) to data
        }.forEach { (rect, data) ->
            paint.color = data?.color ?: return@forEach
            c.drawRect(rect, paint)
        }
    }

    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        val index = parent.getChildAdapterPosition(view)
        Log.d("SideDecoration", "getting offset of index $index")
        val data = dataGetter(index)
        data?.let {
            outRect.left = padding
        }
    }
}

class YourFragment: Fragment() {
    ...

    override fun onViewCreated(view: View, s: Bundle?) {
        ...

        recycler.addItemDecoration(SideDecoration(view.context, dataList::getOrNull))

        ...
    }


    ...
}
```

然而，运行起来后却会发现，如果进行元素移除操作，被删除元素的方块会在动画开始之前消失，整体效果比贯穿分割线没有动画还更难看。

### 问题定位

首先需要来确定，为什么在删除动画开始之前，这个元素的装饰绘制会失效。

梳理一下事件发生顺序：

1. 在数据列表中，删除一个数据；
2. 将删除完数据的列表更新到 `Adapter` 中；
3. `RecyclerView` 响应数据变化，开始执行动画；
4. `ItemDecoration` 在动画过程中不停绘制方块；
   1. 根据 `view` 的位置 `getChildAdapterPosition` ，通过 `dataGetter` 在列表中获取数据；
   2. 根据获取到的数据判断方块信息；

梳理完顺序，应该很快就能发现问题： 在步骤 4.1 中，`dataGetter` 其实是不可能拿到正确的元素的，只可能有两种情况：

* `getChildAdapterPosition` 返回了一个不合法的位置，使得 `dataGetter` 返回了空数据；
* `getChildAdapterPosition` 返回了原来的位置，使得 `dataGetter` 返回了不属于这个卡片的数据。

具体情况，可以使用log大法来判断：

```kotlin
class SideDecoration(context: Context, private val dataGetter: ((Int) -> Data?)) : RecyclerView.ItemDecoration() {
    private val paint = Paint()
    private val padding = context.resources.getDimensionPixelSize(R.dimen.item_side_padding)

    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)
        parent.children.map {
            it to parent.getChildAdapterPosition(it)
        }.map { (view, index) ->
            val data = dataGetter(index)
            Log.d("SideDecoration", "getting data to draw of index $index")
            if (data == null) {
                Log.w(
                    "SideDecoration",
                    "Data null!! view data ${view.findViewById<TextView>(R.id.title).text}"
                )
            }
            view to data
        }.map { (view, data) ->
            RectF(
                0f,
                view.top + view.translationY,
                padding.toFloat(),
                view.bottom + view.translationY
            ) to data
        }.forEach { (rect, data) ->
            paint.color = data?.color ?: return@forEach
            c.drawRect(rect, paint)
        }
    }

    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        val index = parent.getChildAdapterPosition(view)
        Log.d("SideDecoration", "getting offset of index $index")
        val data = dataGetter(index)
        if (data == null) {
            Log.w(
                "SideDecoration",
                "Data null!! view data ${view.findViewById<TextView>(R.id.title).text}"
            )
        }
        data?.let {
            outRect.left = padding
        }
    }
}
```

运行后观察log输出：

> D/MainViewModel: Removing data Data(content=db1c87, color=#ff0000ff at 1
> 
> D/SideDecoration: getting offset of index -1
> 
> W/SideDecoration: Data null!! view data db1c87
> 
> ...
> 
> D/SideDecoration: getting data to draw of index -1
> 
> ...
> 
> W/SideDecoration: Data null!! view data db1c87
> 
> ...

可以看出，运行过程中出现了大量 `getChildAdapterPosition` 返回 -1 的情况，对应的 view 正是被删除数据所绑定的 view 。

查看 `getChildAdapterPosition` 的源代码，发现其最终都是调用到 `androidx.recyclerview.widget.RecyclerView.LayoutParams#getViewAdapterPosition` ，其注释如下：

> Returns the up-to-date adapter position that the view this LayoutParams is attached to
corresponds to.
@return the up-to-date adapter position this view. It may return
{@link RecyclerView#NO_POSITION} if item represented by this View has been removed or
its up-to-date position cannot be calculated.

那么，当数据被删除，对应 `ViewHolder` 被调用 remove 之后，显然就是返回 `RecyclerView#NO_POSITION` ，即 `-1` 了。

从另一个角度考虑， `ViewHolder` 动画是发生在数据删除之后的， `dataGetter` 本身也不可能在新的数据列表之中找到一个被删除的数据给 `ItemDecoration` 进行方块绘制。

### 实现替换

以上问题的根源在于，数据被移除之后不能找到这个被删除数据提供给 `ItemDecoration` 进行方块绘制，那么要考虑的解决思路就是如何将 `ItemDecoration` 所需要的数据传递进去。

回到绘制方法 `override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State)` 中，我们在这里能得到的只有绘制用的画布、 `RecyclerView` 本身，和他相关联的 `state` 状态；进一步扩展的话，还能拿到 `RecyclerView` 当前能展现的子View。

在这里，我们仍然可以考虑将相关的数据存储在 `Fragment` 或者 `ViewModel` 中。但是保存已经被删除的数据并不是一个优雅的操作：

* 并且我们也不知道什么时候可以真正地把他们释放；
* `ItemDecoration` 也没有合适的 key 去准确获取到对应的数据。

因此，还是需要另外寻找存放位置。

考虑到侧边方块是附属于每一个卡片元素的，`RecyclerView` 本身，和他相关联的 `state` 状态并不适合存储这些卡片紧密相关的数据，那么一个合理的存储位置就应当是 `RecyclerView` 当前能展现的子View。

那么如何在一个 `View` 中存储绘制相关数据呢？一个简单朴素的方法就是使用 `android.view.View#setTag(int, java.lang.Object)`：

```kotlin
class SideTagDecoration(context: Context) : RecyclerView.ItemDecoration() {
    private val paint = Paint()
    private val padding = context.resources.getDimensionPixelSize(R.dimen.item_side_padding)

    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        super.onDraw(c, parent, state)
        parent.children.map {
            it to it.getTag(R.id.tag_color) as Int
        }.map { (view, color) ->
            RectF(
                0f,
                view.top + view.translationY,
                padding.toFloat(),
                view.bottom + view.translationY
            ) to ColorUtils.setAlphaComponent(color, (view.alpha * 255).toInt())
        }.forEach { (rect, color) ->
            paint.color = color
            c.drawRect(rect, paint)
        }
    }

    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        super.getItemOffsets(outRect, view, parent, state)
        val content = view.getTag(R.id.tag_content) as String
        if (content.isNotBlank()) {
            outRect.left = padding
        }
    }
}
```

在这个 `ItemDecoration` 的绘制过程中，我们读取了 `ItemView` 里设置的颜色信息，那么就需要在合适的地方设置进去，这里就需要 `ViewHolder` 的协助了：

```kotlin
class ViewHolder(parent: ViewGroup) : RecyclerView.ViewHolder(
    LayoutInflater.from(parent.context).inflate(R.layout.rv_item_layout, parent, false)
) {
    private val bindings = RvItemLayoutBinding.bind(itemView)

    fun bind(data: Data) {
        bindings.root.setTag(R.id.tag_color, data.color)
        bindings.root.setTag(R.id.tag_content, data.content)
        bindings.title.text = data.content
    }
}
```

在这一套实现方案中，我们仅仅将 `ItemDecoration` 所需要的颜色数据存入 `itemView` 的 tag 数据中，保证了额外数据的最小化； `ItemDecoration` 也实现了获取已经被删除的数据（以及正常存在数据）的获取，还节省了一个 lambda 表达式的传入，有着还不错的表现。

## 总结

在平时的开发中，我其实挺少使用 `View.setTag` 方法，认为这个东西比较古老，还可能容易引起内存问题；但是在这个场景下，合理的使用 `setTag` 其实很能快速地解决我们的问题。

示例项目已经在 Github 上开源：[Recyclerview-ani 项目地址](https://github.com/xiaozhikang0916/Recyclerview-ani)。

可以在侧边栏中分别进入普通实现版本 (`Common Decoration`) 和 Tag 实现版本 (`Tag Decoration`) ，点击 `ADD` `REMOVE` 按钮观察其表现。
