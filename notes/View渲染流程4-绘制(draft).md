# View渲染流程4-绘制(draft|TODO)

## 注意

这里只分析硬件加速开启的情况

## 绘制流程分类

目前发现2种绘制场景:
1. frame可以直接处理(不需要sync)
2. frame需要和其他frame一起被处理(需要sync)

## frame的绘制

这个过程会：
1. 构建RenderNode树
2. 确保RenderNode以及对应的录制同步到native，并发起一个绘制frame的请求

```java
// ThreadedRender.java
    void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
        attachInfo.mViewRootImpl.mViewFrameInfo.markDrawStart();
         
        // 构建RenderNode结构
        updateRootDisplayList(view, callbacks);

         // ..

        final FrameInfo frameInfo = attachInfo.mViewRootImpl.getUpdatedFrameInfo();
        // 确保RenderNode以及对应的录制同步到native，并发起一个绘制frame的请求
        // 关于信息的同步什么时候开始的不确定
        // 疑似可能在录制的同时就开始同步了
        int syncResult = syncAndDrawFrame(frameInfo);
        // ..
        }
    }
```

### 绘制操作的录制
TODO


### 录制结果传递给native
TODO


## 非sync场景

frame可以被SurfaceFlinger单独的处理

```text
// 场景举例
view.invalidate()
|
V
frame被绘制到buffer
|
V
buffer为frame构造transaction提交给SurfaceFlinger
```

## sync场景
由于一些原因多个frame需要被SurfaceFlinger同时处理。

### 大致流程
```text
WMS
 ↓
创建同步组
 ↓
向多个跨进程 Window 发起同步绘制
 ↓
各 Window 独立完成绘制并返回结果
 ↓
WMS 等待同步组内所有 Window 完成
 ↓
SurfaceFlinger 在统一时机应用结果
```

SurfaceSyncGroup用于实现同步功能。

SurfaceFlinger 决定参与需要同时处理的frame们的最终处理时机。

## ----------------------

## mReportNextDraw(TODO)

调用点调用reportNextDraw()打上这个标记，performTraversals()中的执行受这个标记的影响(当然这个标记还影响其他地方)。

### mReportNextDraw在performTraversals()中的效果

1. 放宽了performMeasure()/performLayout()的执行条件
2. 在perfomDraw()的最后执行了一些通知机制

### reportNextDraw()在诸多调用点的调用意图

TODO

## sdfad

所以这里是：

draw()收集绘制操作同步给native

native使用我提到的callback告知帧的绘制进度

当进度告知来到以后，需要将金土
