# 事件分发流程2-ViewPostImeInputStage

ViewPostImeInputStage的作用就是将事件传递给View树(传递给mView，也就是DecorView)

## 传递触摸事件给DecorView

```text
//ViewRootImpl
..-> stage_x -> ViewPostImeInputStage -> stage_y -> ..
                        |
                        V
                    onProcess()
                        |
                        V
                 processPointerEvent()    
                        |
                        V
             mView.dispatchPointerEvent(event)        
```

```java
// ViewPostImeInputStage
private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent) q.mEvent;
    boolean handled = mHandwritingInitiator.onTouchEvent(event);

    // 省略代码..

    // If the event was fully handled by the handwriting initiator, then don't dispatch it
    // to the view tree.
    handled = handled || mView.dispatchPointerEvent(event);

    // 省略代码..

    return handled ? FINISH_HANDLED : FORWARD;
}

// DecorView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0 ?
            // 有有效Callback时，优先Callback处理
            cb.dispatchTouchEvent(ev)
            // 其他情况就自己处理
            : super.dispatchTouchEvent(ev);
}
```

### DecorView自己处理

实际会调用DecorView父类ViewGroup#dispatchTouchEvent()处理；这是 **触摸事件进入View树的入口**。


### Callback处理
```text
cb.dispatchTouchEvent(ev)
          |
          V
mWrapped.dispatchTouchEvent(ev)  
```

这里只分析Callback#mWrapper为Activity的情况

```java
// Activity.java中的默认实现
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        // A
        onUserInteraction();
    }
    
    // B
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    
    // C
    return onTouchEvent(ev);
}
```

注释B处优先看window是否处理，如果不处理就交由自己的onToucheEvent()处理。看起来好像事件的处理转移到了window当中？

这里你需要站在Activity的角度来理解：

1. Activity在dispatchTouchEvent()处理事件
2. A/B/C是Activity处理事件的具体方式，并没有转移事件的处理位置。

另外，注释B处实际上会执行到DecorView父类ViewGroup#dispatchTouchEvent()

