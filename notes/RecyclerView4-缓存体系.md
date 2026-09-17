# RecyclerViewx-缓存体系


## 获取vh
可以通过Recycler#getViewForPosition()使用如下优先级缓存体系中获取绑定好的vh，匹配vh的条件使用position或者stableId。
```text
// Recycler#getViewForPosition()
//  |
//  V
// Recycler#tryGetViewHolderForPositionByDeadline()

1. changedScrap(只在preLayout阶段参与查找)
2. attachedScrap
3. hidden views
4. cacheViews
5. ViewCacheExtension
6. RecycledViewPool

7.创建并按需执行vh绑定
```

### 一些细节

如果命中scrap中的vh，调用者需要在对vh进行下一阶段操作以前调用vh.unScrap(),vh此时才会从缓存体系中移除(不然还在mAttachedScrap中存着)。

如果命中hidden views中的vh，这个vh会:
```text
vh -> vh.unhide() -> scrapView(vh) -> 然后从查找中返回
```

## 将vh放入缓存体系

### 直接放入目标位置

```text
// 将view对应的vh放入changedScrap/attachedScrap中
Recycler#scrapView(View view)

// 将view放入hidden体系中
ChildHelper#hide(View view)

// 将vh放入pool中
RecycledViewPool#putRecycledView(ViewHolder scrap)
```

虽然也有mCachedViews.add()可以将vh放入缓存体系，但是这个add()放入只在recycleViewHolderInternal()中调用，这看起来像是和这个方法的逻辑绑定了。

是否能直接放入ViewCacheExtension取决于实现者是否给它提供了对应api。

### 放入api组合使用

这种组合使用可以理解为根据情况将vh放到缓存体系中恰当的位置上。

```text
// 这里是部分例子

//vh根据情况被放入:
// 1.scrap 2.cachedViews 2.RecycledViewPool
LayoutManager#scrapOrRecycleView(Recycler recycler, int index, View view)

//vh根据情况被放入:
// 1.cachedViews 2.RecycledViewPool
Recycler#recycleView(@NonNull View view)

//vh根据情况被放入:
// 1.cachedViews 2.RecycledViewPool
// 另外，只有这个方法中才有cachedViews的放入逻辑
Recycler#recycleViewHolderInternal(ViewHolder holder)
```

## scrap的一些观察

观察源码发现:
1. vh在进行下一步操作之前总是会先调用vh.unScrap()(会尝试从scrap中移除自己)
2. 缓存体系中命中scrap中的vh，返回给调用者以后还未从体系中移除

由此我推断有如下流程：
1. vh进入scrap 
2. 缓存体系命中scrap 
3. 返回给调用者(但是不要移除) 
4. 调用者在执行vh下一步之前调用unScrap()
5. 调用者对vh执行下一步操作

## LinearLayoutManager参与的layout阶段与缓存体系的协作

1. 每次布局之前先将当前vh全部按情况放入缓存体系中(scrap/cachedViews/RecycledViewPool)
2. 然后从缓存体系里拿vh(scrap/hidden views/cachedViews/ViewCacheExtension/RecycledViewPool)出来进行布局
3. RV执行vh动画时会先将vh放入hidden体系(addAnimatingView())，动画结束以后又会从hidden体系中移除(removeAnimatingView)。
4. 执行完动画以后，如果此时scrap中还有vh，RV会将它们看情况放入(cachedViews/RecycledViewPool),并清空scrap


## mViewCacheExtension

### 作用

当Recycler在scrap/hidden views/cache中没找到vh时，会在ViewCacheExtension中进行查找。开发者可以在这里实现自己的缓存逻辑。

### 实现上注意点

1. getViewForPositionAndType()中不能创建View 
2. ViewCacheExtension中缓存的条目需要开发者自己实现加入逻辑

第一点相关注释:
```text
// getViewForPositionAndType()注释节选
This method should <b>not</b> create a new View
```

第二点相关注释:
```text
Note that, Recycler never sends Views to this method to be cached. It is developers responsibility to decide whether they 
want to keep their Views in this custom cache or let the default recycling policy handle it.
```