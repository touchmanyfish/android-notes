# 事件分发流程4-MotionEvent

## 注意

这里只讨论触摸事件

## 结构
```text
MotionEvent
    |
    +-- pointer数组(通过index访问)
    |       |
    |       +-- pointerId（标识触摸点身份）
    |       +-- x/y(使用接受者坐标系)
    |       +-- ...
    |
    +-- actionMasked（发生了什么事件）
    |
    +-- actionIndex（触发事件的pointer在pointer数组中的index）
```

pointer数组中包含了当前应该处理的所有pointer的快照；而actionMasked+actionIndex表示触发当前事件的信息。
```text
MotionEvent = 当前应该处理的所有pointer的快照+触发当前事件的信息
```

### MotionEvent中包含了多少pointer的快照？

不同位置的MotionEvent包含的pointer不同，不过它总是包含它所在位置应该处理的所有pointer的快照。

### actionIndex返回值理解

当ACTION_POINTER_DOWN/ACTION_POINTER_UP时，返回对应index值；其他情况返回0。

## 举例说明

假设手指A按在View上，然后按下了手指B导致MotionEvent产生：
```text
MotionEvent
    |
    +-- pointer数组
    |       |
    |       +-- pointerId[0](对应手指A)
    |               |
    |               +-- x/y
    |               +-- ...
    |       +-- pointerId[1](对应手指B)
    |               |
    |               +-- x/y
    |               +-- ...
    |
    +-- actionMasked(ACTION_POINTER_DOWN)
    |
    +-- actionIndex（1）
```

## 相关api理解


### getPointerId(pointerIndex)

通过index获取pointer数组中对应的pointerId

## getAction()/getActionMasked()/getActionIndex()

action = actionMasked + actionIndex

## getX()/getY()

事件的坐标，使用处理它的View的坐标系。

## getRawX()/getRawY()

事件的坐标，使用屏幕的坐标系。