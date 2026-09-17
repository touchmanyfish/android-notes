# View渲染流程3-布局

## 布局流程

performLayout():
1. 使用测量结果调用根View.layout()对View树进行布局
2. 处理layout()中出现requestLayout()调用的情况

## layout()

这个方法用于对自己进行布局，它大致干了如下几件事：
1. 按需执行onMeasure()
2. 给自己设置postion
3. 按需执行onLayout()来对child进行布局
4. 处理焦点相关

### onLayout()

这个方法是按需调用： 

changed(true) || PFLAG_LAYOUT_REQUIRED存在(TODO)

另外，由于layout()+onLayout()，使得布局操作可以在View树上传递

### requestLayout()
大致作用是尝试触发一次渲染流程

### layout()中出现requestLayout()的情况

处理方式如下：
1. performLayout()中第一次调用mView.layout(),在layout()返回前出现了requestLayout()调用
2. 它们的调用者会被收集起来，对应requestLayout()的实际调用会被推迟到第一次host.layout()执行完毕以后
3. 第一次host.layout()执行完毕
4. 推迟的requestLayout()执行完毕(这些调用会到达ViewRootImpl，但是会跳过scheduleTraversals())
```java
@Override
public void requestLayout() {
    // 这里大致是这样的：
    // 第一次host.layout()
    // mHandlingLayoutInLayoutRequest = true
    // 执行推迟的requestLayout()
    // 这些调用到达这里时由于mHandlingLayoutInLayoutRequest(true)于是跳过scheduleTraversals()
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```
5. 跳过scheduleTraversals()没有关系，这里接着会执行一个测量/布局
```java
// performLayout()代码节选

// 测量
measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight, false /* forRootSizeOnly */);
mInLayout = true;
// 布局
// 第二次调用layout()        
host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

// 看起来这里想表达的是：对于这些推迟的requestLayout()直接就在当前帧处理了，不用再放到后续帧了
```

6. 如果第二次host.layout()中又出现了requestLayout()调用，直接将它们推迟到下一帧处理
```java
// performLayout()代码节选

validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);

if (validLayoutRequesters != null) {
    final ArrayList<View> finalRequesters = validLayoutRequesters;

    // Post second-pass requests to the next frame
    getRunQueue().post(new Runnable() {
        @Override
        public void run() {
            int numValidRequests = finalRequesters.size();

            for (int i = 0; i < numValidRequests; ++i) {
                final View view = finalRequesters.get(i);
                //..
                view.requestLayout();
            }
        }
    });
}
```

#### 总结一下处理

1. 第一次host.layout()调用
2. 在当前帧处理第一次host.layout()中出现的requestLayout()请求
3. 第二次host.layout调用()
4. 在下一帧处理处理第一次host.layout()中出现的requestLayout()请求

#### TODO
```text
// performLayout()中注释节选

// requestLayout() was called during layout.
// If no layout-request flags are set on the requesting views, there is no problem.
// If some requests are still pending, then we need to clear those flags and do
// a full request/measure/layout pass to handle this situation.
```
本小节只分析了requestLayout()的情况，但是注释中提到"layout-request flags",所以本小节的分析不是最精确的。
