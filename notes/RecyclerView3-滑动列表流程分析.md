# RecyclerViewx-滑动列表流程分析

## 滚动流程

```text
           滚动dy到来
              |
              V
LinearLayoutManager#scrollBy()
              |
              V
LinearLayoutManager#fill()(注意此时还未发起真正的滚动):
        1.按需回收即将滑出RV的child
        2.按需布局即将滑入的child
              |
              V
     计算滚动距离scrolled，并开始滚动              
mOrientationHelper.offsetChildren(-scrolled);
```

## 分析场景

```text
// 垂直滚动RV列表，其中E有部分未出现在RV的显示区域(让我们将这部分的长度叫做:E_out_part)
// 此时【向上】滑动的滑动距离dy到来
A
B
C
D
E
```

RV处理如下2个场景：

A: dy == E_out_part

B: dy < E_out_part

C: dy > E_out_part

### 注意
后续讨论忽略了:
* layoutState.mExtraFillSpace
* layoutState.mNoRecycleSpace

## 按需布局逻辑

每种场景添加图例(TODO)

### 场景A

在这种场景没有新的child滑入，所以不执行对新child的布局；不过可能有child滑出，所以会执行回收判断逻辑。


#### mScrollingOffset变化流程

```text
// 在这个场景下
mScrollingOffset == E_out_part == dy
```

#### mAvaliable变化流程
```text
// 在这个场景下
mAvaliable == 0
```

#### 场景A举例

```text
// 发生回收的情况
// 向上滚动dy:20px
// 显然A和B即将滑出RV的可显示区域
// 它们需要被回收
A(1px)
B(19px)
C
D
E(E_out_part:20px)

// 没有回收的情况
// 向上滚动dy:20px
// 显然没有child会滑出显示区域，于是没有回收发生
A(10px)
B
C
D
E(E_out_part:5px)
```

### 场景B

在这种场景没有新的child滑入，所以不执行对新child的布局；不过可能有child滑出，所以会执行回收判断逻辑。

#### mScrollingOffset的理解
```text
mScrollingOffset == E_out_part
             |
             V
mScrollingOffset == dy(fill方法中)             
```

#### mAvaliable的理解

```text
-------E_out_part
dy              |
----滚动终点     |  mScrollingOffset
                |
-------E_out_part

// RV在这种情况下设置
// mAvaliable < 0
mAvaliable = dy - mScrollingOffset
```

#### 相关源码分析
```java
// fill()方法开头部分
if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
    if (layoutState.mAvailable < 0) {
        // 只有场景B才进这段代码
        // 根据前面的分析,这里替换layoutState.mAvailable为前面的结论
        // layoutState.mScrollingOffset = layoutState.mScrollingOffset + (dy - layoutState.mScrollingOffset)
        // layoutState.mScrollingOffset = dy
        layoutState.mScrollingOffset += layoutState.mAvailable;
    }
    
    // 场景A/B都走这段，因为可能会有child滑出
    recycleByLayoutState(recycler, layoutState);
}
```

目前只能将mAvaliable理解为一个能将mScrollingOffset设置为dy都负责值。

#### 场景举例

```text
// dy:10px
A(1px)
B(19px)
C
D
E(E_out_part:20px)

// A即将滑出RV可见区域于是回被回收
// mAvaliable = 10px - 20px
// mScrollingOffset = 20px + (10px - 20px)
```

### 场景C

```text
// 向上滚动dy:100px(整大一点方便讨论)
A
B
C
D
E(E_out_part:20px)
```

在这种场景下当滑动结束以后，只靠当前的child没法填满RV的可显示区域，RV需要在滑动开始前按需填满。RV的做法为：
1. dy到来，计算需要填满的的区域
2. 在fill()方法中依次使用layoutChunk()来用child填；每填一个child立即判断是否有需要回收的child
3. 当区域被填满时停止填充(此时区域可能刚好用完或者child会超出一部分)

#### 流程举例
(TODO:这里画个流程图会更清晰)
```text
// 向上滑动dy(100px)到来


// 再滚动这么多就有新的child滑入了
mScrollingOffset = E_out_part = 20px
// 计算需要填充的区域
mAvaliable = dy - mScrollingOffset = 100px - 20px = 80px
// 检查并回收child


// 填充F(40px)
// 由于F的添加，此时需要滚动过E_out_part，再滚动过F_consumed才会有child滑入
// mScrollingOffset = mScrollingOffset + F_consumed = 60px
// 剩余需要填充的区域
mAvaliable = mAvaliable - 40px = 40px
// 检查并回收child


// 填充G(50px)
// 由于G的添加，此时需要滚动过E_out_part，再滚动过F_consumed，再滚过G_consumed,才会有child滑入
// mScrollingOffset = mScrollingOffset + G_consumed = 110px
// 不过此时G的一部分已经超出了需要填充的区域，RV会将mScrollingOffset的设置为dy
mScrollingOffset =20px(E_out_part) + 80px(需要填充的区域) = 100px(dy)
// 剩余需要填充的区域
mAvaliable = mAvaliable - 50px = -10px
// 检查并回收child
```

##### mScrollingOffset的理解

```text
// mScrollingOffset变化过程
// 滚动这么多以后，再滚有child滑入
mScrollingOffset1 = E_out_part
            |
            V
// 滚动这么多以后，再滚有child滑入
mScrollingOffset2 = E_out_part + F_consumed      
            |
            V           
// 滚动这么多以后，再滚有child滑入           
mScrollingOffset3 = E_out_part + F_consumed + G_consumed
            |
            V
// 滚动这么多以后，再继续滚(mScrollingOffset3 - dy)有child滑入
// dy < mScrollingOffset3表示最后放下的child的一部分超过了需要填充的区域
// dy == mScrollingOffset3表示最后放下的child刚好填满需要填充的区域
mScrollingOffset4 = dy(dy <= mScrollingOffset3)                
```

所以在场景C中mScrollingOffset应该理解为(在大脑里想象这个放入child的过程)：

1. (child未超过滚动终止点)滚动超过mScrollingOffset时一定有child滑入
2. (child超过滚动终止点)滚动超过mScrollingOffset,再继续滚动mScrollingOffset3 - dy才会有有child滑入
3. mScrollingOffset是一个变动的值，它在fill之前+每次放入新child以后会更新，不过会被滚动终止点约束。对于每一次更新后的mScrollingOffset有：mScrollingOffsetX <= dy

#### 注解深入理解

```text
// LayoutState#mScrollingOffset注释
Used when LayoutState is constructed in a scrolling state. It should be set the amount of scrolling we can make without creating a new view. Settings 
this is required for efficient view recycling.
```
上述注释没问题，但是具有误导性，容易误认为是"一旦滑动超过mScrollingOffset则需要添加新的child"，但实际是"一旦滑动超过mScrollingOffset,可能再继续滑一点才需要添加child"，例如：
1. 场景B
2. 场景C中最后一个放下的child部分超过了需要填充的区域

##### mAvaliable的理解

```text
// mAvaliable的变化过程
mAvaliable = dy - E_out_part
           |
           V
mAvaliable = dy - E_out_part - F_consumed
           |
           V
mAvaliable = dy - E_out_part - F_consumed - G_consumed
```

在场景C中mAvaliable表示“需要填充的区域还剩mAvaliable”；当mAvaliable为负时表示最后放置的那个child有一部分超过了需要填充的区域。

## 滑动时的回收逻辑


### 可被回收的child

假设垂直滚动dy到来，它最后导致的实际滚动为dy_real。dy_real <= dy。显然那些bottom在dy_real之上的child已经滑出RV的显示区域了，它们可以被回收。
```text
// A和B可以被回收
A     |dy_real
B     |dy_real
C
D
      |dy_real
      |dy_real
```

### dy_real计算

目前计算dy_real的方案是：依次放置需要的child，然后看它们实际consumed多少空间。(不放置就能提前获取consumed的方案有吗？)

### 直接使用dy_real判断的回收逻辑

依次放置所有所需child，得出dy_real，然后一次性的将bottom在它之前的child给全部回收了。这个方案有如下缺点：

即将不需要的child不能在刚刚的放置过程中被复用；这可能会导致内存分配的增加(没有缓存可用则需要创建child)


```text
假设有100个child都符合dy_real，这些child必须要等到所有需要的child放置完毕以后才会被清理。
```

### RV的回收逻辑

而RV的选择使用一个不断逼近dy_real的值(mScrollingOffset)去判断是否回收，这个值的含义是：

目前不知道最终的dy_real,但是本次至少会滚动此时的mScrollingOffset，先用mScrollingOffset去判断回收，让被回收的child进入缓存流程，如果后续还有需要放置的child，则可以优先使用缓存
而不是创建


## 最终consumed计算

就是计算..


## 发起滚动的逻辑
就是它！
```java
// LinearLayoutManager#scrollBy()
mOrientationHelper.offsetChildren(-scrolled);
```


