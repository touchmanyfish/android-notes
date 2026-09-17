# View渲染流程1-总体流程

## 总体流程

```text
调用点                 ViewRootImpl          Choreographer             系统
  │                         │                       │                    │
  │ ① scheduleTraversals() │                       │                    │
  ├────────────────────────>│                       │                    │
  │                         │                       │                    │
  │                         │ ② 注册 CALLBACK_      │                    │
  │                         │    TRAVERSAL          │                    │
  │                         ├──────────────────────>│                    │
  │                         │                       │                    │
  │                         │                       │ ③ 注册 VSYNC 回调   │
  │                         │                       ├───────────────────>│
  │                         │                       │                    │
  │                         │                       │                    │
  │                         │                       │ ④ VSYNC 通知       │
  │                         │                       │<───────────────────┤
  │                         │                       │                    │
  │                         │                       │ ⑤ 执行 CALLBACK_   │
  │                         │                       │    TRAVERSAL       │
  │                         │                       │                    │
  │                         │<──────────────────────┤                    │
  │                         │ doTraversal()         │                    │
  │                         │     │                 │                    │
  │                         │     ▼                 │                    │
  │                         │ performTraversals()   │                    │
  │                         │                       │                    │
``` 
**流程如下:**

1. 发生了一些事件，需要重新渲染
2. 调用scheduleTraversals()将渲染安排在下一帧到来时
3. 下一帧来了，调用performTraversals()执行渲染流程

## performTraversals()
```text
performTraversals()
│
├── 测量阶段
│   ├── 1. measureHierarchy() × 0..N
│   │       └── performMeasure() × 0..N
│   │
│   └── 2. performMeasure() × 0..N
│
├── 布局阶段
│   └── performLayout() × 0..1
│           └── measureHierarchy() × 0..1
│
└── 绘制阶段
    └── performDraw() × 0..1
```



