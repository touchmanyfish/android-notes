# RecyclerViewx-layout分析:LinearLayoutManager实现

## 注意

后续提到的“lm构造preInfo/postInfo”指的是lm将对应child插到RV上，让RV可以从这些child的信息中构造出Info。

## fill()/layoutChunk()方法

大致逻辑如下：
1. 根据RV计算出一个可用的空间
2. 然后依次将下一个child往RV里放(layoutChunk())，child可以consume可用空间
3. 当可用空间刚好耗尽或者刚刚用超，摆放逻辑结束

## LLM涉及的item分类

* item inserted
* item removed
* item moved
* item changed
* item_in:由于inserted/removed/moved/changed操作导致出现在RV可见区域的item(原本在区域外)
* item_out:由于inserted/removed/moved/changed操作导致离开RV可见区域的item(原本在区域内)

## LLM构造缺失preInfo场景 

### 场景举例

```text
// 例子1:removed(0)，没有新item进入RV显示区域
// RV(100)
A(10)      B(20)
B(20)  ->  C(20)
C(20)      D(60)
D(60)      

// 例子2:removed(0)，新item进入RV显示区域
// RV(100)
A(30)      B(30)
B(30)  ->  C(30)
C(30)      D(30)
D(30)      E(30)//新进入的，缺失preInfo


// 例子3:changed(0)，有新item离开RV显示区域
// RV(100)
A(30)      A(60)
B(30)  ->  B(30)
C(30)      C(30)
D(30)      //D离开了

// 例子4:changed(0)，没有有新item离开RV显示区域
// RV(100)
A(30)      A(20)
B(30)  ->  B(30)
C(30)      C(30)
D(30)      D(30)

// 例子5:changed(0)，新item进入RV显示区域
// RV(100)
A(30)      A(5)
B(30)  ->  B(30)
C(30)      C(30)
D(30)      D(30)
           E(30)//新进入的，缺失preInfo
```

### 构造分析

removed/changed操作可能会出现item_in，它缺失preInfo。LLM的做法是：
1. 当这2个操作时，不管item_in是否会出现，LLM总是在第一次onLayoutChildren()中在列表的尾部预布局下一个child
2. 如果在第二次onLayoutChildren()中发现没item_in没有出现，则会为上一步的child按排一个退场

### 预布局child实现分析

```text
// 例子5:changed(0)，新item进入RV显示区域
// 预布局E
// RV(100)

        第一次onLayoutChildren()      第二次onLayoutChildren() 
A(30)      A(30,not consumed)           A(5)
B(30)  ->  B(30,consumed)        ->     B(30)
C(30)      C(30,consumed)               C(30)
D(30)      D(30,consumed)               D(30)
           E(30,consumed)               E(30) -> 后续去处     
```

RV高度100，原本只能摆放A/B/C/D，结果A选择不consum空间，于是原本被A consum的空间用来摆放E了,通过这样实现了预布局E。

```java
// mIgnoreConsumed默认false

// layoutChunk()
if (params.isItemRemoved() || params.isItemChanged()) {
    // 被removed/changed的item做一个不消耗空间标记
    result.mIgnoreConsumed = true;
}

// fill()
// 跳过可用空间消耗        
if (!layoutChunkResult.mIgnoreConsumed || layoutState.mScrapList != null
        || !state.isPreLayout()) {
    layoutState.mAvailable -= layoutChunkResult.mConsumed;
    remainingSpace -= layoutChunkResult.mConsumed;
}
```

## LLM构造缺失postInfo场景

### 实现

如果非item removed发生了缺失postInfo的情况，LLM会为其构造postInfo。具体做法是：

1. 在第二次onLayoutChildren()中先进行最终布局
2. 然后从next位置依次摆放缺失item对应的child，于是RV就能读取到它们的postInfo

layoutForPredictiveAnimations()方法用于实现第二步。

### 场景举例

```text
// 例子6:moved(9)
// RV(100)

        第一次onLayoutChildren()      第二次onLayoutChildren() 
A(30)      A(30)                         B(30)
B(30)  ->  B(30)        ->               C(30)
C(30)      C(30)                         D(30)
D(30)      D(30)                         E(30)
                                         A(30)// layoutForPredictiveAnimations()中将其放到这个位置
```

## 总体流程

第一次onLayoutChild():
1. 进行旧布局
2. 在旧布局的基础上，按需进行预布局，以此来构造preInfo

第二次onLayoutChild():
1. 进行新布局
2. 按需从next位置开始依次摆放将没有postInfo的item(非item removed)对应的child。