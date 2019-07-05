---
title: "RecyclerView 中 ViewHolder 的异步回调实现"
date: 2019-07-01
tags: 
    - Android
    - RecyclerView
    - Callback
---

在使用 RecyclerView 实现卡片的流布局的页面中，经常会出现卡片内有业务逻辑，有请求接口、查询数据库等异步耗时操作。
有开发同学可能会基于代码复用的考虑，将异步操作的发起和回调写在 ViewHolder 或者 widget 控件里：

```kotlin
class Holder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    val button: BizButton
    var bizData: BizData

    init {
        button.setOnClickListener {
            //在工作线程里发起网络请求，在请求返回后调用 onBizSuccess()
            button.startBiz(bizData, object: BizListener {
                override fun onBizSuccess() {
                    bizData.doSomeThing()
                    updateUI()
                }
            })
        }
    }

    fun bind(data: BizData) {
        bizData = data
    }

    fun updateUI() {
        //to update you view holder
    }
}
```

这样的逻辑可能在简单的自测中运行良好，并且 BizButton 的业务逻辑也有一定的复可用能力，实属不可多得的好代码。但是异步操作的存在给代码引入了问题。

## 藏着的坑

### 1. 异步操作返回后，页面还存在吗

在上述代码里 BizListener 在异步操作完成后直接调用 ViewHolder 的 updateUI() 方法去更新界面。
但是在调用更新时可能面临着 ViewHolder 被回收、页面不可见或被销毁、 fragment 被调用 onDestroyView() 而使得 Adapter 被置空的问题，直接调用更新容易引起 NPE 。

聪明的朋友可能马上就想到了，需要做保护：

```kotlin
button.startBiz(
    bizData, object: BizListener {
        override fun isCancel(): Boolean {
            //比如一系列 fragment 是否存活 activity 是否存活 view 是否可见的判断
            return !itemView.isShown
        }
        override fun onBizSuccess() {
            if (!isCancel()) {
                bizData.doSomeThing()
                updateUI()
            }
        }
    }
)
```

在 isCancel() 的保护之下，页面销毁之后回调不会再去尝试更新数据或者ui，在页面确实被用户关闭的情况下不会有问题，但是嵌套在 `FragmentPagerAdapter` 的 fragment 会有另一个问题：
> 如果回调在 fragment 被移出可视区域后被调用，后续更新操作将被取消。
> fragment 重新可见后将仍然显示旧的数据。

此问题在现有框架下打补丁暂时无解。

### 2. 异步操作返回后，你的 ViewHolder 还是你的吗

上述代码中的 BizListener 实际是作为匿名内部类实现的，持有着外部 ViewHolder 的引用，
因此能在 onBizSuccess() 里直接调用 ViewHolder 的更新方法。

但是要注意到 ViewHolder 是用在 RecyclerView 里的， RecyclerView 的复用功能可能发生在异步操作的等待过程中，会出现的是：

> A 数据的业务回调`a`的 onBizSuccess() 被调用时，`a`持有的 ViewHolder 绑定的数据不是 A 。

在回调时直接更新，如果此时 ViewHolder 已经被复用，那么会导致一个错误的数据被更新、界面上一个错误的ui元素被修改。

跟上一题同样聪明的朋友可能又想到了，要做保护：

```kotlin
button.startBiz(
    bizData, object: BizListener {
        override fun isSameData(data: BizData): Boolean {
            data == this@Holder.bizData
        }
        override fun isCancel(): Boolean {
            //比如一系列 fragment 是否存活 activity 是否存活 view 是否可见的判断
            return !itemView.isShown
        }
        override fun onBizSuccess(data: BizData) {
            if (!isCancel() && isSameData(data)) {
                bizData.doSomeThing()
                updateUI()
            }
        }
    }
)
```

然而这只是在 ViewHolder 被复用之后放弃更新，需要更新的数据仍然保留着旧值，并且没有下一次更新的机会。

同样，这个问题无法在 ViewHolder 内部逻辑（例如，不依赖于 Adapter ）解决。

## 问题根源

对于上面提到的问题，有着相似的根本原因。

### ViewHolder 没有完整的生命周期

ViewHolder 本身是没有生命周期的回调的，即不能通过实现原生接口的方式关注 ViewHolder 在 RecyclerView 中的状态变化。

> 因此，也不要在 ViewHolder 内部注册/反注册 Observer 。

Adapter 和 LayoutManager 中可能有类似于生命周期的调用，但是不够完善，是否被调用也不被保证，对解决问题1帮助不大。

这也引申出第二点：

### ViewHolder 不应该有业务逻辑，应只根据所承载的数据作展现

上面两个问题的直接原因在于，耗时操作的回调不知道 应不应该 / 在哪个时候 / 更新哪一个数据，但究其根源，**ViewHolder 不应该去尝试更新数据， 而是仅作为数据展现的载体**，去展现*已经被更新好的数据*。

## 解决思路

耗时操作、业务逻辑等代码不应该实现在 ViewHolder 或者更里层的 widget 控件里，那么应当实现在什么地方？

* 从 ViewHolder 数据承载的功能出发，那么逻辑应该写在**数据源的持有者**里，通过更新数据来通知 ViewHolder 更新ui；
* 从生命周期的角度考虑，拥有完整生命周期的页面宿主，如`Fragment`和`Activity`，或者它们所持有的`ViewModel`应当是最恰当的位置。

为了最大程度地利用生命周期，可以选择使用`LiveData`来实现回调通知。

于是搓出新一版的代码：

```kotlin
data class BizData(
    val id: Int,
    var data: String
)

interface BizObserver {
    fun onSuccess(biz: Map<Int, BizData>)
}

class BizManager(
    lifecycleOwner: LifecycleOwner,
    observer: BizObserver
) {
    private var latestVersion = 0
    private val pending: MutableMap<Int, Pair<BizData, Int>> = TreeMap()
    private val liveData = MutableLiveData<Map<Int, Pair<BizData, Int>>>()
    init {
        liveData.observe(lifecycleOwner, ObserverWrapper(observer))
    }

    fun startBiz(biz: BizData) {
        NetWorkBiz.startBiz(biz, object: BizListener {
            override onSuccess() {
                postData(biz)
            }
        })
    }

    fun postData(biz: BizData) {
        latestVersion += 1
        pending[biz.id] = Pair(biz, latestVersion)
        liveData.postValue(pending)
    }
}

class ObserverWrapper(private val ob: BizObserver) : Observer<Map<Int, Pair<BizData, Int>>> {
    private var currentVersion = 0

    override fun onChanged(t: Map<Int, Pair<BizData, Int>>?) {
        t?.let { map ->
            var latestVersion = currentVersion
            val success = mutableMapOf<Int, BizData>()
            fun append(list: MutableMap<Int, BizData>, pair: Pair<BizData, Int>) {
                if (pair.second > currentVersion) {
                    list[pair.first.id] = pair.first
                    lastestVersion = Math.max(latestVersion, pair.second)
                }
            }
            map.values.forEach {
                append(success, it)
            }
            ob.onSuccess(success)
            currentVersion = latestVersion
        }
    }
}

class YourFragment : Fragment {
    lateinit var bizManager: BizManager
    var adapter: BizAdapter? = null
    var data: MutableList<BizData>? = null
    lateinit var recyclerView: RecyclerView
    private val ob = object : BizObserver {
        override fun onSuccess(biz: Map<Int, BizData>) {
            DiffUtil.calculateDiff(object : DiffUtil.Callback() {
                override fun areItemsTheSame(oldIndex: Int, newIndex: Int): Boolean {
                    return oldIndex == newIndex
                }

                override fun getOldListSize(): Int {
                    return data?.size ?: 0
                }

                override fun getNewListSize(): Int {
                    return oldListSize
                }

                override fun areContentsTheSame(oldIndex: Int, newIndex: Int): Boolean {
                    val oldBiz = data?.get(newIndex) ?: return true
                    val newBiz = biz[oldBiz.id] ?: return true
                    if (oldBiz == newBiz) {
                        oldBiz.data = newBiz.data
                        return false
                    }
                    return true
                }
            }).dispatchUpdatesTo(adapter ?: return)
        }
    }
    override  fun onCreate(savedInstanceState: Bundle?) {
        bizManager = BizManager(this, ob)
    }

    override fun onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        //初始化你的布局
        adapter = BizAdapter(this)
    }
    override fun onDestroyView() {
        super.onDestroyView()
        recyclerView.adapter = null
        mAdapter = null
    }

    fun doBiz(index: Int) {
        bizManager.startBiz(data[index])
    }
}

class BizAdapter(private fragment: YourFragment) {
    override onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        return BizHolder(itemView, fragment)
    }
}

class Holder(itemView: View, f: YourFragment) : RecyclerView.ViewHolder(itemView) {
    val button: BizButton
    var bizData: BizData

    init {
        button.setOnClickListener {
            f.doBiz(adapterPosition)
        }
    }

    fun bind(data: BizData) {
        bizData = data
        // Only do the ui things
    }

    fun updateUI() {
        //to update you view holder
    }
}
```

这样就可以实现卡片流页面里耗时操作的回调，同时不会因为需要更新的卡片被回收复用、或者页面宿主不可见等情况导致的数据更新失败了。
