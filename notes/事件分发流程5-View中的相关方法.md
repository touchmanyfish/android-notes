# 事件分发流程5-View中的相关方法

## View#dispatchTouchEvent()

处理优先级：
1. mOnTouchListener.onTouch()
2. View#onTouchEvent()

## View#onTouchEvent()

处理优先级：

1. mTouchDelegate.onTouchEvent()
2. click/longClick等