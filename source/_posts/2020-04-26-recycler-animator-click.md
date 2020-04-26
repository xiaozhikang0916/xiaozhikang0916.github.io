---
title: RecyclerView 的 item 在动画过程中的点击事件
date: 2020-04-26 16:42:45
tags:
    - Android
    - RecyclerView
    - ItemAnimator
---

在项目中应用了 MVVM 模式之后，就能享受到 `DiffUtil` 带来的计算最小变动集的便利性，以及在列表项更新时能用上自带动画。

但是因为页面更新应用了动画，使得页面响应点击时出现了问题。

## 问题现象

在我的页面中，列表每一项都带有一个订阅按钮，对应数据有一个 “是否已经订阅” 的字段。每次点击按钮时，都根据目前数据是否已经订阅，向服务器发送订阅或者取消订阅的请求，然后根据请求的返回结果是否成功，使用 `DiffUtil` 更新当前页面列表项。

示例如下：

```kotlin
class ItemViewHolder(itemView: View, private val onCheck: ((Int, Boolean) -> Unit)? = null) :
    RecyclerView.ViewHolder(itemView) {
    private var data: Item? = null
    private val titleView = itemView.findViewById<TextView>(R.id.title)
    private val checkButton = itemView.findViewById<TextView>(R.id.button)

    init {
        checkButton.setOnClickListener(::onClick)
    }

    private fun onClick(view: View) {
        onCheck?.invoke(adapterPosition, data?.checked ?: false)
    }

    fun bind(bindData: Item) {
        data = bindData
        titleView.text = bindData.title
        checkButton.isSelected = bindData.checked
    }
}

data class Item(
    val id: Long,
    val title: String,
    var checked: Boolean = false
)
```

代码看起来很美好，但是测试反馈过来说：快速多次点击按钮时，会重复发出完全一样的请求，而后续的请求则毫无疑问地失败了。

## 问题分析

得知问题后，马上开始开始着手分析。经过不停地断点调试和输出 log 之后（过程省略），会发现连续点击时，响应点击事件的 View 及其 ViewHolder 都一直是同一个对象；在替换动画播放完成后再次点击，响应点击事件的就变成了一个新的 View 了。

继续深入研究， RecyclerView 的动画默认是在 `DefaultItemAnimator` 中实现的。 而对于拥有相同 id 的数据，仅仅是其中的部分数据有更新，没有进行位置变换的话， `DiffUtil` 会调用 adapter 的 `notifyItemChanged` 方法通知更新。 `Change` 的动画在 `DefaultItemAnimator` 中的实现可以参见方法 `animateChangeImpl` ：

```java
void animateChangeImpl(final ChangeInfo changeInfo) {
    final RecyclerView.ViewHolder holder = changeInfo.oldHolder;
    final View view = holder == null ? null : holder.itemView;
    final RecyclerView.ViewHolder newHolder = changeInfo.newHolder;
    final View newView = newHolder != null ? newHolder.itemView : null;
    if (view != null) {
        final ViewPropertyAnimator oldViewAnim = view.animate().setDuration(
                getChangeDuration());
        mChangeAnimations.add(changeInfo.oldHolder);
        oldViewAnim.translationX(changeInfo.toX - changeInfo.fromX);
        oldViewAnim.translationY(changeInfo.toY - changeInfo.fromY);
        oldViewAnim.alpha(0).setListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animator) {
                dispatchChangeStarting(changeInfo.oldHolder, true);
            }
            @Override
            public void onAnimationEnd(Animator animator) {
                oldViewAnim.setListener(null);
                view.setAlpha(1);
                view.setTranslationX(0);
                view.setTranslationY(0);
                dispatchChangeFinished(changeInfo.oldHolder, true);
                mChangeAnimations.remove(changeInfo.oldHolder);
                dispatchFinishedWhenDone();
            }
        }).start();
    }
    if (newView != null) {
        final ViewPropertyAnimator newViewAnimation = newView.animate();
        mChangeAnimations.add(changeInfo.newHolder);
        newViewAnimation.translationX(0).translationY(0).setDuration(getChangeDuration())
                .alpha(1).setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationStart(Animator animator) {
                        dispatchChangeStarting(changeInfo.newHolder, false);
                    }
                    @Override
                    public void onAnimationEnd(Animator animator) {
                        newViewAnimation.setListener(null);
                        newView.setAlpha(1);
                        newView.setTranslationX(0);
                        newView.setTranslationY(0);
                        dispatchChangeFinished(changeInfo.newHolder, false);
                        mChangeAnimations.remove(changeInfo.newHolder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }
}
```

可以得知， `DefaultItemAnimator` 对于 change 的动画实现是将旧的 viewHolder 的不透明度 (alpha) 渐变到0，将新的 viewHolder 的不透明度渐变到1。呈现出来的效果便是在同一个位置，旧 viewHolder 淡出，被淡入的新 viewHolder 替换。

而上述点击的问题就出在动画播放过程中，点击事件被即将消失的旧 viewHolder 捕获了。

## 解决方案

在知道问题原因之后，就需要开始找解决办法了。既然问题原因出在动画播放过程中的点击响应，那么解决的思路应该是：在动画播放过程中不接受点击事件的分发，在动画播放完成后恢复响应。对此我想出了几个解决的思路。

### 业务方处理点击的生效和禁用

最开始的思路是，既然点击事件的监听是在 ViewHolder 中设置的，那么在需要禁用和启用对应事件的时候，由 ViewHolder 自行判断就好了。然而紧接着遇到的问题是， ViewHolder 能收到的外部通知仅仅是 adapter 中的 `onBindViewHolder` ，它在动画开始前就被调用，而动画完成后没有任何事件通知，因此无法实现点击事件的响应恢复。

另一个问题在于，如果将这一步交给业务方的 ViewHolder 自行实现，将会出现大量的模版代码，不利于项目的维护。

基于这两个原因，这一个思路连实验代码都没有写便被我放弃了。

### Animator 中自动设置是否响应点击

既然问题出在动画过程中，而系统也为我们开放了动画管理的接口，那么很自然地会想到自己复写动画的开始和结束，对应设置点击的启用与否。

在上面的 `animateChangeImpl` 代码中，可以看到第 15 行和第 37 行分别调用了 `dispatchChangeStarting` ，并且将当前传入的 viewHolder 是新是旧也作为参数传入，正适合我们进行控制，马上开始尝试：

ClickAnimator.kt：

```kotlin
class ClickItemAnimator : DefaultItemAnimator() {
    override fun onChangeStarting(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        super.onChangeStarting(item, oldItem)
        item?.itemView?.isClickable = false
    }

    override fun onChangeFinished(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        super.onChangeFinished(item, oldItem)
        item?.itemView?.isClickable = true
    }
}
```

在我的设想中，动画开始时直接禁用整个 itemView 的是否可点击属性，在动画播放完成后将其恢复就能实现想要的效果。然而实际运行起来发现问题依旧，经过分析可知 `isClickable` 属性仅对单个 view 有效，其中的子 view 不受影响，而再次尝试 `enable` 属性也是如此，那么便只能尝试将其子 view 都进行设置了：

```kotlin
class ClickItemAnimator : DefaultItemAnimator() {
    override fun onChangeStarting(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        super.onChangeStarting(item, oldItem)
        item.itemView?.let { setClickable(it, false) }
    }

    override fun onChangeFinished(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        super.onChangeFinished(item, oldItem)
        item.itemView?.let { setClickable(it, true) }
    }

    private fun setClickable(view: View, clickable: Boolean) {
        view.isClickable = clickable
        if (view is ViewGroup) {
            view.children.filter { it.visibility == View.VISIBLE }.forEach { setClickable(it, clickable) }
        }
    }
}
```

然而在实际运行前便想到了另一个问题： itemView 中不一定每一个子 view 都是需要响应点击的，我们不应该在动画完成后遍历设置，否则可能会破坏其原有的业务功能；暂存其原本的属性也不现实，会需要维护一个巨大的缓存池，并且维护其与实际的 itemView 的对应关系也需要巨大的精力。

#### 为 itemView 嵌套自定义 View ，按需拦截点击事件

如果不可以遍历设置 itemView 中的点击属性，一个折衷的办法可以是自定义一个 ViewGroup ，响应 Animator 的设置，按需拦截传入的点击事件。

然而这个办法需要修改 itemView 的根布局，不适合接入到已有的项目中；而且多一层布局嵌套可能会对绘制性能产生影响，因此这个方法没有进行尝试验证。

### Animator 分发事件，业务自行响应

既然上述两个思路都有各自的缺陷而不可行，那么可以将两个方案组合起来各取优点使用：由 Animator 进行动画的开始、结束事件分发，各业务的 ViewHolder 按需进行响应的接入，达到按需响应、改动量小的目标：

```kotlin
class ClickItemAnimator : DefaultItemAnimator() {

    override fun onChangeStarting(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        (item as? OnItemAnimationListener)?.onItemAnimationStatus(false)
        super.onChangeStarting(item, oldItem)
    }

    override fun onChangeFinished(item: RecyclerView.ViewHolder?, oldItem: Boolean) {
        (item as? OnItemAnimationListener)?.onItemAnimationStatus(true)
        super.onChangeFinished(item, oldItem)
    }
}

interface OnItemAnimationListener {
    fun onItemAnimationStatus(shouldClickEnabled: Boolean)
}

class ClickItemViewHolder(itemView: View, onCheck: ((Int, Boolean) -> Unit)) :
    ItemViewHolder(itemView, onCheck), OnItemAnimationListener {
    private var handleClick = true

    override fun onClick(view: View) {
        onCheck?.takeIf { handleClick }?.invoke(adapterPosition, data?.checked ?: false)
    }

    override fun onItemAnimationStatus(shouldClickEnabled: Boolean) {
        handleClick = shouldClickEnabled
    }

    override fun bind(bindData: Item) {
        handleClick = true
        super.bind(bindData)
    }
}
```

在这里，需要在动画过程中禁用点击的 viewHolder 自行实现接口 `OnItemAnimationListener` ，根据其传入参数设置标记位；在实际点击事件中判断标记位属性值，来决定是否需要发起业务逻辑。这样需要修改的代码量少，同时也不会对原有的业务逻辑产生影响。

## 示例demo

我把上面的问题与解决做成了一个 demo 项目：[ItemAnimatorBlockClick](https://github.com/xiaozhikang0916/ItemAnimatorBlockClick)。

在页面底部点击 `Default Fragment` 按钮可以显示原本有问题的页面，点击一个 `Check` 按钮后，对应列表项将会开始替换动画（动画时间被延长到 1.5s 方便观察）；在动画过程中再次点击同一个按钮，将会看到一个 `error` 的 log 输出，因为传入的状态与数据中存有的状态不符，并且按钮动画消失，马上转变回原有的样子。

`Click Fragment` 按钮可以显示经过修复的页面，重复上述操作会发现在动画过程中，重复的点击事件不会被响应，也不会看到 `error` 的 log 输出，直到动画播放完成后才能正确响应下一次点击。

## 后文

在项目中，大部分页面都直接调用了 `adapter.notifyDataSetChanged` 方法，直接刷新整个页面的数据。这样做确实有其便利性，不用考虑在什么地方具体发生了怎样的更新、不用担心动画执行的时间里发生问题。然而尽可能地使用到系统原生提供的工具和动画支持，也是提高用户体验的一个方向，在遇到问题 -> 解决问题的过程中，自己也能得到学习提高的机会。
