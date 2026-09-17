# 事件分发流程3-ViewGroup#dispatchTouchEvent()

## 注意

后面讨论中的 **“某个pointer的事件序列”** 指“和这个pointer的DOWN事件相关的所有事件”

另外，这里只讨论触摸事件。

## 主要逻辑

ViewGroup#dispatchTouchEvent()方法实现了事件分发逻辑，主要干了如下3件事

* 事件拦截处理 
* 确定事件序略归属
* 根据归属结果分发事件

## 确定事件序列归属

事件序列归属有如下两种情况：

1. 所有pointer的事件序列只归属于一个拥有者(split off)
2. 多个pointer对应的事件序列可以归属于同一个拥有者，并且可以同时存在多个这样的拥有者(split on)

无论哪种情况，当ViewGroup获得事件序列归属时它会获得所有pointers的事件序列归属！

### split touch

split指的是“split touch”，它控制了事件序列归属的粒度。

### 事件序列的拥有者

pointer_x对应事件序列会被传递给拥有者:

* 拥有者为ViewGroup:super.dispatchTouchEvent()
* 拥有者为child:child.dispatchTouchEvent()

### child归属候选者要求
```text
// 基本条件
1. !child.canReceivePointerEvents()
// pointer需要按在这个child上
2. !isTransformedTouchPointInView(x, y, child, null)
```

### 归属判定:split off

判定流程如下：
1. 第一个DOWN事件到来
2. 遍历child中的候选者，谁先消费DOWN谁就拥有所有的事件序列归属
3. 2中没有child拥有者诞生，ViewGroup获得所事件序列归属

```text
// 举例
// split on
// ViewGroup包含A/B两个View

1. pointer_1按在A上
2. pointer_x按在了ViewGroup的任何地方，事件都会传递给A
```

### 归属判定:split on

判定流程如下：
1. pointer_x_DOWN事件到来
2. 遍历child中的候选者，谁先消费pointer_x_DOWN谁就拥pointer_x的事件序列归属
3. 2中没有child拥有者诞生,将point_x的事件序列归属到第一个成为拥有者的child
4. 如果3中没有拥有者诞生,ViewGroup获得所事件序列归属

另外，如果ViewGroup获得了第一个pointer的事件序列，则ViewGroup会获得所有pointer的事件序列(注释A)
```java
// ViewGroup#dispatchTouchEvent()
// Check for interception.
final boolean intercepted;

if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
// 省略代码...
} else {
    // 注释A
    intercepted = true;
}
```

#### 例子：split on-child第一次获得事件序列

```text
// split on，ViewGroup包含A/B两个View

pointer_1按在A上，A获得pointer_1的事件序列
```

#### 例子：split on-child拥有者再次获得事件序列

```text
// split on，ViewGroup包含A/B两个View

pointer_2按在A上，A获得pointer_2的事件序列
```

#### 例子：split on-没有child候选者获得事件序列，将事件序列归属于第一个child拥有者

```text
// split on，ViewGroup包含A/B两个View

1.pointer_1按在A上
2.pointer_2按在ViewGroup但是A/B之外的地方，pointer_2事件序列归属于A
```

#### 例子：split on-ViewGroup获取所有序列

```text
// split on，ViewGroup包含A/B两个View

1. pointer_1按在ViewGroup上但是A/B之外的区域
2. 任何pointer_x按在ViewGroup上任意区域，它们的的事件序列归都属于ViewGroup
```


### TouchTarget

在前面讨论的归属逻辑中，显然需要一个地方来维护“拥有者-它拥有的事件序列(一个或者多个pointer的)”。TouchTarget负责实现这个维护。

```java
//TouchTarget

// 归属拥有者
public View child;

// 拥有者拥有的事件序列对应的pointers
public int pointerIdBits;

// split on时可能同时存在多个拥有者，这里用链表维护
public TouchTarget next;
```

前面提到，当ViewGroup作为事件序列拥有者时它会拥有所有pointers的事件序列，所以ViewGroup不用TouchTarget来帮其维护与事件序列的关系。

### CANCEL事件

## 根据归属结果分发事件
```java
// ViewGroup.java
private boolean dispatchTransformedTouchEvent(
        MotionEvent event, 
        boolean cancel, 
        View child, 
        int desiredPointerIdBits
)
```

当事件序列归属确定好以后使用上述方法进行事件分发

### child

表示分发的target，如果为null表示分发给自己

### 坐标转换(TODO)

event中的坐标使用ViewGroup的坐标系，当分发给target之前需要转成target的坐标系(转换逻辑没理解TODO)

### cancel

表示在在target发送event时会将其修改为CANCEL事件

### desiredPointerIdBits
举例说明
```text
// event包含pointer_1/pointer_2/pointer_3相关信息
// child已经获得pointer_2/pointer_3事件序列的归属
// desiredPointerIdBits中包含pointer_2+pointer_3
// dispatchTransformedTouchEvent()中会将pointer_2+pointer_3的相关信息拆出来(必要时)传递给child
```

## 事件拦截处理
```java
// ViewGroup#dispatchTouchEvent()
// Check for interception.
final boolean intercepted;

if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    // 注释B:ViewGroup【未获得】所有pointers的事件序列时
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }

} else {
    // 注释A：ViewGroup【已获得】所有pointers的事件序列时
    intercepted = true;
}
```

### 拦截分析：已获得

注释A为了实现前面提到的：

ViewGroup获得了第一个pointer的事件序列，则ViewGroup会获得所有pointer的事件序列

### 拦截分析：未获得

此时的处理流程为：

1. 是否有disallowIntercept标记(requestDisallowInterceptTouchEvent())
2. 自己是否拦截(onInterceptTouchEvent())
3. 拦截成功功，将当前事件转成CANCEL，发送给当前所有的child事件拥有者

下面举例说明：

1. 收到MOVE事件，假设此时有1个事件序列拥有者child1
2. MOVE拦截成功
3. 将MOVE转成CANCEL发送给child1，并产生一个处理结果handle
4. ViewGroup将handle向上传递
5. 从下一个事件开始的后续事件都会发送给ViewGroup

步骤3中child1如果返回hanlde(false),看起来好像没有消费这个事件，于是ViewGroup可以继续消费。然而在ViewGroup#dispatchTouchEvent()中:

1. 拦截事件发生时的那个event永远不会被ViewGroup消费
2. 它会被转成CANCEL事件发给当前所有的child事件拥有者