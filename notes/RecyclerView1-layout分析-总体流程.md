# RecyclerViewx-layout分析:总体流程

## 视角

RV和LayoutManager抽象类(不涉及它的具体实现)

## 注意

后续提到的“lm构造preInfo/postInfo”指的是lm将对应child插到RV上，让RV可以从这些child的信息中构造出Info。

## RV的layout过程

```text
1.从界面读取preInfo
2.第一次lm.layoutChildren()
3.再读一下界面新增的preInfo
4.第二次lm.layoutChildren()
5.从界面读取postInfo
6.执行所有相关predictive animation = f(preInfo?, oldInfo?)
```

### predictive animation实现面临的问题

当执行notify操作时，RV会读取item对应vh在RV执行notify之前(preInfo)和之后的状态(postInfo)，然后根据它们来执行动画:
```text
predictive animation = f(preInfo?,postInfo?)；
```

然而在某些notify操作下(不止这一种)，item可能没有preInfo或者postInfo；例如：
```text
// 假设我们要实现如下列表(注意，这里不是在说LLM!这里是一种逻辑上的讨论)

// 例子Z
// 没有preInfo的情况
// notifyItemInserted(0)插入Z
A         Z
B   ->    A
C         B
D         C
          D(RV可显示区域以外) 

// 对于Z，它没有preInfo

// 例子A
// 没有postInfo的情况
// notifyItemRemoved(0)删除A
A         B
B   ->    C
C         D
D    

// 对于A，它没有postInfo
```

对于这个问题，RV通过2次调用来将手动构造原本不存在的info的职责通过2次onLayoutChildren()调用传递给了LayoutManager.

### 构造preInfo/postInfo

当child在RV上时就可以被读取出Info(ItemHolderInfo)，所以lm在为child构造Info时，只需要根据自己的逻辑将它们添加到RV上即可。

### lm.onLayoutChildren()在第一次调时的职责

LayoutManager需要在第一次onLayoutChildren()调用时为自己感兴趣(假设当前有m种缺失场景，可以只处理其中n种，n<=m)的preInfo缺失的情况构造preInfo。

### 读取preInfo

分两次读取:
```text
1.从界面读取preInfo
2.第一次lm.layoutChildren()
3.再读一下界面新增的preInfo(注意，这里不会再次读取步骤1中已经读取过的)
```

preInfos = 直接就存在的preInfo(第一次读) + lm选择性构造的preInfo(第二次读)

**TODO**: 为什么不在第一次lm.layoutChildren()以后统一读取preInfo?

### lm.onLayoutChildren()在第二次调时的职责

LayoutManager需要在第二次onLayoutChildren()调用时:
1. 进行真正的布局
2. 为自己感兴趣(假设当前有m种缺失场景，可以只处理其中n种，n<=m)的postInfo缺失的情况构造postInfo。

### 读取postInfo

直接从界面读postInfo = 直接就存在的postInfo + lm选择性构造的postInfo

### predictive animation执行
```text
predictive animation = f(preInfo?, oldInfo?)
```
其中的"?"表示对应的Info不存在，LayoutManager也没有手动构造。

```java
// RecyclerView#dispatchLayoutStep3()

// 开始处理动画
mViewInfoStore.process(mViewInfoProcessCallback);
```

### “predictive animation”如何翻译

如果直接翻译成“预测动画”，感觉不对，毕竟一个不存在的东西怎么预测？不如粗略的翻译成：
```text
基于新旧状态的动画，如果状态不存在lm可以根据自己的实现选择性的构造状态
```


