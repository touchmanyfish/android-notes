# View渲染流程5-invalidate(draft)

## TODO

invalidate() 的核心作用是标记 View 的绘制内容需要更新，并将这个需求向父层传播，最终由 ViewRootImpl 调度下一次 scheduleTraversals()。